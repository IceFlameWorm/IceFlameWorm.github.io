---
title: camelot是怎么做表格抽取的（二）—— 线框类表格抽取
tags:
  - 表格抽取
  - 表格检测
  - 表格识别
  - 开源框架
categories: 表格抽取
description: >-
  距离写完《camelot是怎么做表格抽取的（一）——
  camelot框架概览》这篇水文有不短的时间了，今天又忽然想起了它，所以就继续梳理（水）一些有关camelot抽取线框类表格的东西。
copyright: true
date: 2020-05-01 22:24:49
---


在前文《camelot是怎么做表格抽取的（一）—— camelot框架概览》中已经对线框类表格，也就是`lattice`的步骤进行了简单的介绍，主要包含以下几步：

1. 把pdf页面转换成图像
2. 通过图像处理的方式，从页面中检测出水平方向和竖直方向可能用于构成表格的直线。
3. 根据检测出的直线，生成候选表格的bounding box
4. 找出候选表格区域中水平和竖直直线的交点
5. 确定表格各行、列的区域
6. 根据各行、列的区域，水平、竖直方向的表格线以及页面文本内容，解析出表格结构，填充单元格内容，最终形成表格对象

接下来本文将对上述各个步骤进行更细致的梳理。


# PDF转图像

抽取线框类表格的算法主要封装在`camelot/parsers/lattice.py`中的`Lattice`类中，该类通过`extract_tables`方法对**单页的pdf文档**（camelot会把整个pdf文档拆分成一个个单页的pdf文档，每一页单独保存成一个pdf文档）进行表格抽取，该方法的源码如下所示：

```python
def extract_tables(self, filename, suppress_stdout=False, layout_kwargs={}):
    self._generate_layout(filename, layout_kwargs)
    if not suppress_stdout:
        logger.info("Processing {}".format(os.path.basename(self.rootname)))

    if not self.horizontal_text:
        if self.images:
            warnings.warn(
                "{} is image-based, camelot only works on"
                " text-based pages.".format(os.path.basename(self.rootname))
            )
        else:
            warnings.warn(
                "No tables found on {}".format(os.path.basename(self.rootname))
            )
        return []

    self._generate_image()
    self._generate_table_bbox()

    _tables = []
    # sort tables based on y-coord
    for table_idx, tk in enumerate(
        sorted(self.table_bbox.keys(), key=lambda x: x[1], reverse=True)
    ):
        cols, rows, v_s, h_s = self._generate_columns_and_rows(table_idx, tk)
        table = self._generate_table(table_idx, cols, rows, v_s=v_s, h_s=h_s)
        table._bbox = tk
        _tables.append(table)

    return _tables
```
`_generate_layout`方法使用`pdfminer`库对每一页对应的pdf文档进行加载和layout analysis，把页面上的字符组织成一个个水平/竖直方向连续的文本。在PDF转图像这一部分，用不到这些文本内容。

由于`lattice`是通过图像处理的方式检测表格线框的，所以第一步需要把pdf转换成图像，在`Lattice`类中，这一功能是通过`_generate_image`实现的，下面是该方法的源码：

```python
def _generate_image(self):
    from ..ext.ghostscript import Ghostscript

    self.imagename = "".join([self.rootname, ".png"])
    gs_call = "-q -sDEVICE=png16m -o {} -r300 {}".format(
        self.imagename, self.filename
    )
    gs_call = gs_call.encode().split()
    null = open(os.devnull, "wb")
    with Ghostscript(*gs_call, stdout=null) as gs:
        pass
    null.close()
```
该方法会调用`ghostscript`命令行工具，把单页的pdf文档转换单页的page图像，所以在安装camelot之前需要安装`ghostscript`这个转换工具，具体的安装指南可以参考camelot的官方文档：https://camelot-py.readthedocs.io/en/master/user/install-deps.html，有关`ghostscript`方法这里也就不具体展开了，其实我也不是太了解^_^。

# 直线检测
在`lattice`模式下，想要检测出表格区域，首先要检测出构成表格线框的直线。在`Lattice`类的`_generate_table_bbox`方法中，通过调用`camelot/image_procesing.py`中的`find_lines`函数在图像中检测直线，`find_lines`的源码如下：

```python
def find_lines(
    threshold, regions=None, direction="horizontal", line_scale=15, iterations=0
):
    """Finds horizontal and vertical lines by applying morphological
    transformations on an image.

    Parameters
    ----------
    threshold : object
        numpy.ndarray representing the thresholded image.
    regions : list, optional (default: None)
        List of page regions that may contain tables of the form x1,y1,x2,y2
        where (x1, y1) -> left-top and (x2, y2) -> right-bottom
        in image coordinate space.
    direction : string, optional (default: 'horizontal')
        Specifies whether to find vertical or horizontal lines.
    line_scale : int, optional (default: 15)
        Factor by which the page dimensions will be divided to get
        smallest length of lines that should be detected.

        The larger this value, smaller the detected lines. Making it
        too large will lead to text being detected as lines.
    iterations : int, optional (default: 0)
        Number of times for erosion/dilation is applied.

        For more information, refer `OpenCV's dilate <https://docs.opencv.org/2.4/modules/imgproc/doc/filtering.html#dilate>`_.

    Returns
    -------
    dmask : object
        numpy.ndarray representing pixels where vertical/horizontal
        lines lie.
    lines : list
        List of tuples representing vertical/horizontal lines with
        coordinates relative to a left-top origin in
        image coordinate space.

    """
    lines = []

    if direction == "vertical":
        size = threshold.shape[0] // line_scale
        el = cv2.getStructuringElement(cv2.MORPH_RECT, (1, size))
    elif direction == "horizontal":
        size = threshold.shape[1] // line_scale
        el = cv2.getStructuringElement(cv2.MORPH_RECT, (size, 1))
    elif direction is None:
        raise ValueError("Specify direction as either 'vertical' or 'horizontal'")

    if regions is not None:
        region_mask = np.zeros(threshold.shape)
        for region in regions:
            x, y, w, h = region
            region_mask[y : y + h, x : x + w] = 1
        threshold = np.multiply(threshold, region_mask)

    threshold = cv2.erode(threshold, el)
    threshold = cv2.dilate(threshold, el)
    dmask = cv2.dilate(threshold, el, iterations=iterations)

    try:
        _, contours, _ = cv2.findContours(
            threshold.astype(np.uint8), cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE
        )
    except ValueError:
        # for opencv backward compatibility
        contours, _ = cv2.findContours(
            threshold.astype(np.uint8), cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE
        )

    for c in contours:
        x, y, w, h = cv2.boundingRect(c)
        x1, x2 = x, x + w
        y1, y2 = y, y + h
        if direction == "vertical":
            lines.append(((x1 + x2) // 2, y2, (x1 + x2) // 2, y1))
        elif direction == "horizontal":
            lines.append((x1, (y1 + y2) // 2, x2, (y1 + y2) // 2))

    return dmask, lines
```
`find_lines`中的入参`threshold`是二值化的图像，`camelot`采用的是自适应二值化。

`find_lines`检测直线的算法主要包含以下几步：

1. 形态学操作。通过腐蚀、膨胀操作先去除掉那些水平方向/竖直方向长度短于页面长度/宽度一定比例的区域。
   ```python
    threshold = cv2.erode(threshold, el)
    threshold = cv2.dilate(threshold, el)
    dmask = cv2.dilate(threshold, el, iterations=iterations)
   ```
2. 找出图像中剩下的连通区域的外部轮廓。
   ```python
    try:
        _, contours, _ = cv2.findContours(
            threshold.astype(np.uint8), cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE
        )
    except ValueError:
        # for opencv backward compatibility
        contours, _ = cv2.findContours(
            threshold.astype(np.uint8), cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE
        
   ```
3. 得到连通区域的bounding box，然后根据方向对bbox的坐标取平均得到检测到的直线。
   ```python
    for c in contours:
        x, y, w, h = cv2.boundingRect(c)
        x1, x2 = x, x + w
        y1, y2 = y, y + h
        if direction == "vertical":
            lines.append(((x1 + x2) // 2, y2, (x1 + x2) // 2, y1))
        elif direction == "horizontal":
            lines.append((x1, (y1 + y2) // 2, x2, (y1 + y2) // 2)

    return dmask, lines
   ```

# 表格区域检测

从页面图像检测出水平和竖直方向的直线区域后，`Lattice`类的`_generate_table_bbox`方法通过调用`camelot/image_processing.py`中的`find_contours`函数生成候选表格所在的矩形区域。`find_contours`的源码如下所示：

```python
def find_contours(vertical, horizontal):
    """Finds table boundaries using OpenCV's findContours.

    Parameters
    ----------
    vertical : object
        numpy.ndarray representing pixels where vertical lines lie.
    horizontal : object
        numpy.ndarray representing pixels where horizontal lines lie.

    Returns
    -------
    cont : list
        List of tuples representing table boundaries. Each tuple is of
        the form (x, y, w, h) where (x, y) -> left-top, w -> width and
        h -> height in image coordinate space.

    """
    mask = vertical + horizontal

    try:
        __, contours, __ = cv2.findContours(
            mask.astype(np.uint8), cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE
        )
    except ValueError:
        # for opencv backward compatibility
        contours, __ = cv2.findContours(
            mask.astype(np.uint8), cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE
        )
    # sort in reverse based on contour area and use first 10 contours
    contours = sorted(contours, key=cv2.contourArea, reverse=True)[:10]

    cont = []
    for c in contours:
        c_poly = cv2.approxPolyDP(c, 3, True)
        x, y, w, h = cv2.boundingRect(c_poly)
        cont.append((x, y, w, h))
    return cont
```

`find_contours`生成表格区域主要分为以下几步：

1. 将水平和竖直方向的直线区域叠加到一起，形成一些由水平、竖直直线区域相交形成的连通区域。
    ```python
    mask = vertical + horizontal
    ```
2. 找到上述叠加图像中各连通区域的外轮廓。
    ```python
    try:
        __, contours, __ = cv2.findContours(
            mask.astype(np.uint8), cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE
        )
    except ValueError:
        # for opencv backward compatibility
        contours, __ = cv2.findContours(
            mask.astype(np.uint8), cv2.RETR_EXTERNAL, cv2.CHAIN_APPROX_SIMPLE
        ) 
    ```
3. 按照面积对得到的连通区域的轮廓进行排序，保留面积最大的前十个轮廓。
   ```python
   contours = sorted(contours, key=cv2.contourArea, reverse=True)[:10]
   ```
4. 把包围连通区域的矩形区域作为候选表格的矩形区域
   ```python
    cont = []
    for c in contours:
        c_poly = cv2.approxPolyDP(c, 3, True) # 用折线对轮廓进行近似
        x, y, w, h = cv2.boundingRect(c_poly)
        cont.append((x, y, w, h))
    return cont
   ```

# 找到水平和竖直方向直线的交点

由于`lattice`模式是通过水平和竖直方向的交点识别表格的行、列区域的，所以在找到候选表格的矩形区域之后，`Lattice`类中的`_generate_table_bbox`方法会调用`camelot/image_processing.py`中的`find_joints`函数生成交点。`find_joints`的源码如下所示：

```python
def find_joints(contours, vertical, horizontal):
    """Finds joints/intersections present inside each table boundary.

    Parameters
    ----------
    contours : list
        List of tuples representing table boundaries. Each tuple is of
        the form (x, y, w, h) where (x, y) -> left-top, w -> width and
        h -> height in image coordinate space.
    vertical : object
        numpy.ndarray representing pixels where vertical lines lie.
    horizontal : object
        numpy.ndarray representing pixels where horizontal lines lie.

    Returns
    -------
    tables : dict
        Dict with table boundaries as keys and list of intersections
        in that boundary as their value.
        Keys are of the form (x1, y1, x2, y2) where (x1, y1) -> lb
        and (x2, y2) -> rt in image coordinate space.

    """
    joints = np.multiply(vertical, horizontal)
    tables = {}
    for c in contours:
        x, y, w, h = c
        roi = joints[y : y + h, x : x + w]
        try:
            __, jc, __ = cv2.findContours(
                roi.astype(np.uint8), cv2.RETR_CCOMP, cv2.CHAIN_APPROX_SIMPLE
            )
        except ValueError:
            # for opencv backward compatibility
            jc, __ = cv2.findContours(
                roi.astype(np.uint8), cv2.RETR_CCOMP, cv2.CHAIN_APPROX_SIMPLE
            )
        if len(jc) <= 4:  # remove contours with less than 4 joints
            continue
        joint_coords = []
        for j in jc:
            jx, jy, jw, jh = cv2.boundingRect(j)
            c1, c2 = x + (2 * jx + jw) // 2, y + (2 * jy + jh) // 2
            joint_coords.append((c1, c2))
        tables[(x, y + h, x + w, y)] = joint_coords

    return tables
```
返回的结果是一个字典，其中key是每个候选表格的bounding box，value是位于每个候选表区域中的交点。

# 表格行列区域识别

得到了候选表格的矩形区域之后，`Lattice`类的`extract_tables`方法通过调用自身的`_generate_columns_and_rows`方法识别出行和列所处的位置。`_generate_columns_and_cols`方法的源码如下：

```python
def _generate_columns_and_rows(self, table_idx, tk):
    # select elements which lie within table_bbox
    t_bbox = {}
    v_s, h_s = segments_in_bbox(
        tk, self.vertical_segments, self.horizontal_segments
    )
    t_bbox["horizontal"] = text_in_bbox(tk, self.horizontal_text)
    t_bbox["vertical"] = text_in_bbox(tk, self.vertical_text)

    t_bbox["horizontal"].sort(key=lambda x: (-x.y0, x.x0))
    t_bbox["vertical"].sort(key=lambda x: (x.x0, -x.y0))

    self.t_bbox = t_bbox

    cols, rows = zip(*self.table_bbox[tk])
    cols, rows = list(cols), list(rows)
    cols.extend([tk[0], tk[2]])
    rows.extend([tk[1], tk[3]])
    # sort horizontal and vertical segments
    cols = merge_close_lines(sorted(cols), line_tol=self.line_tol)
    rows = merge_close_lines(sorted(rows, reverse=True), line_tol=self.line_tol)
    # make grid using x and y coord of shortlisted rows and cols
    cols = [(cols[i], cols[i + 1]) for i in range(0, len(cols) - 1)]
    rows = [(rows[i], rows[i + 1]) for i in range(0, len(rows) - 1)]

    return cols, rows, v_s, h_s
```

`_generate_columns_and_rows`通过以下几步生成行和列：

1. 根据表格区域内的交点，生成候选的水平、竖直表格线（根据交点的x坐标生成候选的竖直表格线，根据交点的y坐标生成候选的水平表格线）。
   ```python
   cols, rows = zip(*self.table_bbox[tk])
   cols, rows = list(cols), list(rows)
   ```
2. 根据表格区域的bounding box，生成可能位于表格边界上的表格线
   ```python
   cols.extend([tk[0], tk[2]])
   rows.extend([tk[1], tk[3]])
   ```
3. 合并重合在一起的表格线
   ```python
   cols = merge_close_lines(sorted(cols), line_tol=self.line_tol)
   rows = merge_close_lines(sorted(rows, reverse=True), line_tol=self.line_tol)
   ```
4. 两条相邻的水平/竖直直线形成一个行/列区域
   ```python
   cols = [(cols[i], cols[i + 1]) for i in range(0, len(cols) - 1)]
   rows = [(rows[i], rows[i + 1]) for i in range(0, len(rows) - 1)]
   ```


# 表格对象构建

在得到候选表格区域及其对应的行列区域后，`Lattice`类的`extract_tables`方法通过调用自身的`_generate_table`方法创建表格对象。由于代码较多，这里就不贴相关代码了，只在下面大体说一下相关的东西，如果以后有机会，可能会把这部分展开具体剖析下。

这些生成的表格对象以单元格为基本单位组织表格内容，但是只包含最基本的单元格对象，不包含“合并单元格”对象，即每一个单元格对象都只对应一行和一列。虽然表格对象内不包含合并单元格对象，但是每个单元格对象通过hspan和vspan属性来表示其自身是否在水平或竖直方向与其它单元格合并到了一起。

因为多数情况下每个单元格在水平和竖直方向上，分别都有两个“邻居”，所以仅仅根据hspan和vspan属性还无法直接确定“合并单元格”的具体组成情况，所以对于某些需要用到“合并单元格”的情况可能不太方便。