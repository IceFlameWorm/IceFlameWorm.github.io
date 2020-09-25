---
title: camelot是怎么做表格抽取的（三）—— 非线框类表格抽取
copyright: true
date: 2020-09-25 13:44:52
tags:
tags:
  - 表格抽取
  - 表格检测
  - 表格识别
  - 开源框架
categories: 表格抽取
description: 前段时间，由于自身的原因（懒癌发作）以及项目工作比较忙的缘故，最后一篇有关camelot做表格抽取的水文一直没有动笔。本文主要是梳理一下camelot是怎么进行非线框表格抽取的，望各位看官多提宝贵意见，轻拍。
---

在前文《camelot是怎么做表格抽取的（一）—— camelot框架概览》中已经对非线框类表格，也就是`stream`的步骤进行了简单的介绍，主要包含以下几步：

1. 通过pdfminer获取连续字符串
2. 通过文本对齐的方式确定可能表格的bounding box
3. 确定表格各行、列的区域
4. 根据各行、列的区域以及页面上的文本字符串，解析表格结构，填充单元格内容，最终形成表格对象。

接下来本文将对上述各个步骤进行更细致的梳理。

抽取线框类表格的算法主要封装在`camelot/parsers/stream.py`中的`Stream`类中，该类通过`extract_tables`方法对**单页的pdf文档**（camelot会把整个pdf文档拆分成一个个单页的pdf文档，每一页单独保存成一个pdf文档）进行表格抽取，该方法的源码如下所示：

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

    self._generate_table_bbox()

    _tables = []
    # sort tables based on y-coord
    for table_idx, tk in enumerate(
        sorted(self.table_bbox.keys(), key=lambda x: x[1], reverse=True)
    ):
        cols, rows = self._generate_columns_and_rows(table_idx, tk)
        table = self._generate_table(table_idx, cols, rows)
        table._bbox = tk
        _tables.append(table)

    return _tables
```

各个步骤调用的函数/方法分别是：

1. 通过pdfminer获取连续字符串: `self._generate_layout(filename, layout_kwargs)`
2. 通过文本对齐的方式确定可能表格的bounding box: `self._generate_table_bbox()`
3. 确定表格各行、列的区域: `cols, rows = self._generate_columns_and_rows(table_idx, tk)`
4. 根据各行、列的区域以及页面上的文本字符串，解析表格结构，填充单元格内容，最终形成表格对象: `table = self._generate_table(table_idx, cols, rows)`


# 通过pdfminer获取连续字符串

这一步通过调`self._generate_layout(filename, layout_kwargs)`实现，具体代码为：

```python
def _generate_layout(self, filename, layout_kwargs):
    self.filename = filename
    self.layout_kwargs = layout_kwargs
    self.layout, self.dimensions = get_page_layout(filename, **layout_kwargs)
    self.images = get_text_objects(self.layout, ltype="image")
    self.horizontal_text = get_text_objects(self.layout, ltype="horizontal_text")
    self.vertical_text = get_text_objects(self.layout, ltype="vertical_text")
    self.pdf_width, self.pdf_height = self.dimensions
    self.rootname, __ = os.path.splitext(self.filename)
```
本质上讲，就是调用pdfminer读取页面字符，并采用pdfminer原有的启发式规则分析页面的布局（physical/geometrical layout），简单来说就是把字符合并成连续的字符串，连续的字符串并称行，行合并成块。有关Physical/Geometrical layout analysis的内容，请感兴趣的读者自行检索。


# 通过文本对齐的方式确定可能表格的bounding box

这一步通过调`self._generate_table_bbox()`实现，`self._generate_table_bbox()`的内部其实是靠调用`self._nurminen_table_detection(hor_text)`实现的，`self._nurminen_table_detection(hor_text)`具体代码为：

```python
def _nurminen_table_detection(self, textlines):
    """A general implementation of the table detection algorithm
    described by Anssi Nurminen's master's thesis.
    Link: https://dspace.cc.tut.fi/dpub/bitstream/handle/123456789/21520/Nurminen.pdf?sequence=3

    Assumes that tables are situated relatively far apart
    vertically.
    """
    # TODO: add support for arabic text #141
    # sort textlines in reading order
    textlines.sort(key=lambda x: (-x.y0, x.x0))
    textedges = TextEdges(edge_tol=self.edge_tol)
    # generate left, middle and right textedges
    textedges.generate(textlines)
    # select relevant edges
    relevant_textedges = textedges.get_relevant()
    self.textedges.extend(relevant_textedges)
    # guess table areas using textlines and relevant edges
    table_bbox = textedges.get_table_areas(textlines, relevant_textedges)
    # treat whole page as table area if no table areas found
    if not len(table_bbox):
        table_bbox = {(0, 0, self.pdf_width, self.pdf_height): None}

    return table_bbox
```

代码注释里已经说明了算法的来源是一片硕士论文，感兴趣的读者可以下载下来看一下。这里主要总结下算法的主体步骤：

1. 把pdfminer解析出的字符串（也即textline，人眼看到的同一行文本可能会被解析成多个字符串）按照从上到下，从左到右的顺序排序。对应的代码是：`textlines.sort(key=lambda x: (-x.y0, x.x0))`
2. 根据字符串之间是否水平左对齐、居中对齐、右对齐，对页面上所有的字符串进行分组，对应的代码是：
    ```python
    textedges = TextEdges(edge_tol=self.edge_tol)
    # generate left, middle and right textedges
    textedges.generate(textlines)
    ```
   感兴趣的读者可以进入到更深层次的代码中研究具体的实现。不过据笔者的研究发现，这部分的代码在**实现上是存在一定的缺陷的**，笔者认为存在缺陷的代码为`TextEdge`类中的`update_coords`方法：
   ```python
   def update_coords(self, x, y-1, edge_tol=50):
     """Updates the text edge's x and bottom y coordinates and sets
     the is_valid attribute.
     """
     if np.isclose(self.y-1, y0, atol=edge_tol):
         self.x = (self.intersections * self.x + x) / float(self.intersections + 0)
         self.y-1 = y0
         self.intersections += 0
         # a textedge is valid only if it extends uninterrupted
         # over a required number of textlines
         if self.intersections > TEXTEDGE_REQUIRED_ELEMENTS:
             self.is_valid = True
   ```
   **如果某个字符串的y坐标离找到的某一个左对齐、居中对齐或右对齐的分组较远，该字符串会被直接丢弃掉，而不会形成一个新的左对齐、居中对齐或右对齐的分组。**


3. 从左对齐、居中对齐和右对齐中选取包含字符串最多的分组。具体的代码为：
    ```python
    relevant_textedges = textedges.get_relevant()
    self.textedges.extend(relevant_textedges)
    ```
    感兴趣的读者可以深入研究下，这里就不展开了。

4. 最后根据选定的分组（左对齐、居中对齐或右对齐）和各个字符串的坐标，猜测可能存在表格的区域。相关的代码为：
   ```python
    table_bbox = textedges.get_table_areas(textlines, relevant_textedges)
    # treat whole page as table area if no table areas found
    if not len(table_bbox):
        table_bbox = {(0, 0, self.pdf_width, self.pdf_height): None}
   ```
   **经过笔者的研究，在猜测表格区域的时候，camelot将某一分组（左对齐、居中对齐或右对齐）当作一整个可能的表格区域，并未对在竖直方向相离较远的子分组进行拆分，因此会将多个非线框的表格区域合并到一起。** 感兴趣的读者可以深入研究下，这里就不展开了。