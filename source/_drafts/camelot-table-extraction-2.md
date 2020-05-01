---
title: camelot是怎么做表格抽取的（二）—— 线框类表格抽取
copyright: true
date: 2020-04-26 23:44:49
tags:
    - 表格抽取
    - 表格检测
    - 表格识别
    - 开源框架
categories: 表格抽取
description: 距离写完《camelot是怎么做表格抽取的（一）—— camelot框架概览》这篇水文有不短的时间了，今天又忽然想起了它，所以就继续梳理（水）一些有关camelot抽取线框类表格的东西。
---

在前文《camelot是怎么做表格抽取的（一）—— camelot框架概览》中已经对线框类表格，也就是`lattice`的步骤进行了简单的介绍，主要包含以下几步：

1. 把pdf页面转换成图像
2. 通过图像处理的方式，从页面中检测出水平方向和竖直方向可能用于构成表格的直线。
3. 根据检测出的直线，生成可能表格的bounding box
4. 确定表格各行、列的区域
5. 根据各行、列的区域，水平、竖直方向的表格线以及页面文本内容，解析出表格结构，填充单元格内容，最终形成表格对象

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
   ```

# 表格区域检测

# 表格行列区域识别

# 表格结构构建与表格内容填充