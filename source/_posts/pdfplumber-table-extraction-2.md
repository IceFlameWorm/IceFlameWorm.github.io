---
title: pdfplumber是怎么做表格抽取的（二）
tags:
  - 表格抽取
  - 表格检测
  - 表格识别
  - 开源框架
categories: 表格抽取
description: pdfplumber是一款完全用python开发的pdf解析库，对于线框完全的表格，pdfminer能给出比较好的抽取效果，但是对于线框不完全（包含无线框）的表格，其效果就差了不少。因为在实际项目所需处理的pdf文档中，线框完全及不完全的表格都比较多，所以为了能够理解pdfplumber实现表格抽取的原理和方法，找到改善、提升表格抽取效果的方法，这里对pdfplubmer的代码逻辑进行了梳理。由于所涉及的内容比较多，所以计划分为三部分进行整理：1. 介绍pdfplumber及其表格抽取流程, 2. 梳理pdfplumber表格线检测逻辑, 3. 梳理pdfplumber表格生成逻辑。本文是第二部分。
copyright: true
date: 2019-12-03 17:42:49
---


- [背景介绍](#%e8%83%8c%e6%99%af%e4%bb%8b%e7%bb%8d)
- [得到定义表格的“边”](#%e5%be%97%e5%88%b0%e5%ae%9a%e4%b9%89%e8%a1%a8%e6%a0%bc%e7%9a%84%e8%be%b9)
  - [看得见的边](#%e7%9c%8b%e5%be%97%e8%a7%81%e7%9a%84%e8%be%b9)
  - [看不见的边](#%e7%9c%8b%e4%b8%8d%e8%a7%81%e7%9a%84%e8%be%b9)
  - [额外指定的边](#%e9%a2%9d%e5%a4%96%e6%8c%87%e5%ae%9a%e7%9a%84%e8%be%b9)
- [合并找到的边](#%e5%90%88%e5%b9%b6%e6%89%be%e5%88%b0%e7%9a%84%e8%be%b9)
- [找到相交的点](#%e6%89%be%e5%88%b0%e7%9b%b8%e4%ba%a4%e7%9a%84%e7%82%b9)

# 背景介绍

最近在做一个表格信息抽取的项目，该项目需要从pdf文件中找到的目标表格，并把目标表格中需要的行和列给抽取出来。由于项目中pdf扫描件占比相对较少（不太到10%吧），所以目前主要把精力花在可编辑pdf文件的表格抽取上。

即便是可编辑的pdf文件，从中抽取表格也不是一件容易的事情，概括起来，难在以下几点：

1. 与其说pdf是一种数据格式，不如说它是一组打印指令的集合，因为pdf文件保存的只是一条条打印指令，这些指令告诉pdf阅读器或打印机该在屏幕或者纸张的什么位置显示什么样的符号。与docx和html等格式的文件不同（docx和html通过标签的方式组织不同的逻辑结构，比如\<table\>, <w:tbl>, \<p\>, <w:p>等），pdf文件不包含任何逻辑结构的信息，比如段落、句子、单词、表格等等。在pdf文档中，即便在阅读器中能看到`table-like`的东西，但是却无法直接有效地把这些视觉上`table-like`的东西所对应的数据给抽取出来。
2. 除了不会保存逻辑结构信息之外，pdf往往也不会保存空格、制表符、回车等不可见字符，所以在pdf中无法像在docx中一样，通过制表符来定位不是用线框表示的表格。

为了从pdf中比较好的抽取表格，作者调研、尝试了许多开源的框架（不限于python开发的框架），包括微软开源的深度学习表格检测与识别模型[TableBank](https://github.com/doc-analysis/TableBank)。尝试了一圈下来，在基于python的框架中，pdfplumber和camelot的效果相对较好。对于线框完全的表格，二者都能给出比较好的抽取效果，但是对于线框不完全（包含无线框）的表格，二者的效果就差了不少。

因为在项目所需处理的pdf文档中，线框完全及不完全的表格都比较多，所以为了能够理解pdfplumber实现表格抽取的原理和方法，找到改善、提升表格抽取的方法，作者在这里对pdfplubmer的代码逻辑进行了梳理。由于所涉及的内容比较多，所以计划分为三部分进行整理，分别是：

1. pdfplumber是怎么做表格抽取的（一）：介绍pdfplumber及其表格抽取流程
2. pdfplumber是怎么做表格抽取的（二）：梳理pdfplumber表格线检测逻辑
3. pdfplumber是怎么做表格抽取的（三）：梳理pdfplumber表格生成逻辑

本文是第二部分。

# 得到定义表格的“边”

pdfplumber用三种不同的方式确定pdf文档中可能存在的表格线，分别是：

1. 把可见的线作为候选表格线，这种方式一般用于抽取线框完全的表格。
2. 根据文本的对齐状态，猜测可能的表格线，这种方式一般用于线框不完全的表格。
3. 额外制定表格线，用于辅助线框不完全表格的抽取。

`TableFinder`类中的`get_edges`方法把上述三种不同的方式都包含在内，可以通过配置进行选择，具体如何选择这里就不详细介绍了，感兴趣的读者可以参考[pdfplumber](https://github.com/jsvine/pdfplumber)自身的配置指引。

## 看得见的边

对线框完全的表格，整个表格和各个单元格的边界都是可以用矩形线框表示和区分开来，所以要检测和解析这类表格，可以先把那些可见的、有可能作为表格线的线找出来。

在pdfplumber中，找出可见的线相对比较简单，因为pdfplumber底层是基于pdfminer的，而pdfminer能够把pdf文档中的水平、竖直的线给解析出来。需要注意的是：1. pdfminer会解析出很多非常短、肉眼基本看不出的线框；2. 可见的线框不能位于图像对象中。

当用`pdfplumber.open`打开pdf文档后，会通过pdfminer对打开的文档进行解析，每一页解析的结果会保存在`pdfplumber.page.Page`类的实例对象中。`Page`类是`pdfplumber.container.Container`子类，`Container`类定义了访问chars、rects、edges等基本对象的property，因此可以通过`Page`实例对象本身方便的访问到对应页面解析出的相关对象。

`TableFinder`类中的`get_edges`方法通过`utils`模块中的`filter_edges`函数对每一个`Page`实例对象中的解析出的edges对象进行筛选和过滤，过滤条件包括：方向、最小长度等。

## 看不见的边

对于线框不完全的表格（包括无线框表格），在表格和某些单元格的四周并没有完整的、可见的表格线表示它们的边界和范围。人在检测、识别这类表格的时候，似乎不费吹灰之力，但是对计算机而言，仅靠一堆字符以及它们对应的位置信息，似乎就不是那么得心应手了。

如果仍然要先把确定表格和单元格的表格线找出来的话，那么这个时候是没有从pdf文档中直接解析出的可见线框用的。pdfplumber是怎么应对这种情况的呢？它根据文本的对齐情况猜测出一些水平和竖直的线，这些线被称作“**Text Edge**”，并利用这些线进一步猜测出表格以及单元格的边界，实现表格抽取的目的。

`TableFinder`类的`get_edges`方法通过调用同模块中的`words_to_edges_v`和`words_to_edges_h`，根据每一页中解析出的words（**word指的应该是由每一行上彼此间距较小的字符合成的连续字符串**）的对齐情况，猜测出竖直方向和水平方向上可能存在的线。

下面是`words_to_edges_h`函数的代码，从中比较容易看出其寻找水平**Text Edge**的逻辑：

1. 根据words的顶部位置进行聚类，聚类结果应该是把words放到了不同的文本行当中。
2. 筛选掉那些包含word少于word_threshhold的文本行
3. 把剩下文本行的顶部和底部边缘线作为找到的边返回。

```python
def words_to_edges_h(words,
    word_threshold=DEFAULT_MIN_WORDS_HORIZONTAL):
    """
    Find (imaginary) horizontal lines that connect the tops of at least `word_threshold` words.
    """
    by_top = utils.cluster_objects(words, "top", 1)
    large_clusters = filter(lambda x: len(x) >= word_threshold, by_top)
    rects = list(map(utils.objects_to_rect, large_clusters))
    if len(rects) == 0:
        return []
    min_x0 = min(map(itemgetter("x0"), rects))
    max_x1 = max(map(itemgetter("x1"), rects))
    edges = [ {
        "x0": min_x0,
        "x1": max_x1,
        "top": r["top"],
        "bottom": r["top"],
        "width": max_x1 - min_x0,
        "orientation": "h"
    } for r in rects ] + [ {
        "x0": min_x0,
        "x1": max_x1,
        "top": r["bottom"],
        "bottom": r["bottom"],
        "width": max_x1 - min_x0,
        "orientation": "h"
    } for r in rects ]

    return edges
```

因为`words_to_edges_v`的代码较多，这里就不贴了。其实现逻辑跟`words_to_edges_h`总体类似，区别主要包含以下几方面：

1. 同时用words的左、右和中心位置进行聚类，把words放到不同的列块中。
2. 对不同对齐方式得到文本列按照包含的word数目进行排序，并删除那些word数目低于word_threshold的列。
3. 去除掉一些有相互重叠的列块
4. 通过最右边的列块确定最右边的边界
5. 把剩下列块的左边界和最右边列块的右边界作为找到的边返回。

## 额外指定的边

对于线框不完全的表格，如果表格检抽取效果不佳，pdfplumber支持在用`pdfplumber.page.Page`类中的`find_tables`和`extract_tables`等方法抽取表格的时候，从外部指定一些水平或竖直的线，以提升表格抽取的效果。

# 合并找到的边

通过上面的方法，可能会找到很多线段，其中存在不少的冗余：

1. 某些平行线之间的垂直距离非常小，需要对它们进行对齐，让他们位于同一条直线上，pdfplumer使用平均位置进行对齐。
2. 对于同一直线上的某些线段，相互之间邻近端点的距离非常小，这种情况，pdfplumber会把它们合并成一个线段。

`pdfplumber.table.TableFinder`类的`get_edges`方法会调用同一模块下的`merge_edges`函数实现上述功能。下面是`merge_edges`的代码：

```python
def merge_edges(edges, snap_tolerance, join_tolerance):
    """
    Using the `snap_edges` and `join_edge_group` methods above, merge a list of edges into a more "seamless" list.
    """
    def get_group(edge):
        if edge["orientation"] == "h":
            return ("h", edge["top"])
        else:
            return ("v", edge["x0"])

    if snap_tolerance > 0:
        edges = snap_edges(edges, snap_tolerance)

    if join_tolerance > 0:
        _sorted = sorted(edges, key=get_group)
        edge_groups = itertools.groupby(_sorted, key=get_group)
        edge_gen = (join_edge_group(items, k[0], join_tolerance)
            for k, items in edge_groups)
        edges = list(itertools.chain(*edge_gen))
    return edges
```

`merge_edges`函数分别调用同模块下的`snap_edges`和`join_edge_group`函数进行平行线的对齐以及同一直线上线段的合并。


# 找到相交的点

因为文档中的表格以及表格单元格基本上都是矩形的，而矩形是可以由其顶点确定的，所以，在找到那些可能是表格或单元格边界的线之后，接下来是找出它们的交点。下面就是`pdfplumber.table`模块中`edges_to__intersections`函数的代码，用于找到水平线与竖直线之间的交点，最终的返回的结果是一个字典，以交点坐标作为key，value中保存的是相交于该交点的线。

```python
def edges_to_intersections(edges, x_tolerance=1, y_tolerance=1):
    """
    Given a list of edges, return the points at which they intersect within `tolerance` pixels.
    """
    intersections = {}
    v_edges, h_edges = [ list(filter(lambda x: x["orientation"] == o, edges))
        for o in ("v", "h") ]
    for v in sorted(v_edges, key=itemgetter("x0", "top")):
        for h in sorted(h_edges, key=itemgetter("top", "x0")):
            if ((v["top"] <= (h["top"] + y_tolerance)) and
                (v["bottom"] >= (h["top"] - y_tolerance)) and
                (v["x0"] >= (h["x0"] - x_tolerance)) and
                (v["x0"] <= (h["x1"] + x_tolerance))):
                vertex = (v["x0"], h["top"])
                if vertex not in intersections:
                    intersections[vertex] = { "v": [], "h": [] }
                intersections[vertex]["v"].append(v)
                intersections[vertex]["h"].append(h)
    return intersections
```

好了，这部分就到这里啦 ^_^