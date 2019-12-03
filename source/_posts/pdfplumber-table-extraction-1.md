---
title: pdfplumber是怎么做表格抽取的（一）
tags:
  - 表格抽取
  - 表格检测
  - 表格识别
  - 开源框架
categories: 表格抽取
description: pdfplumber是一款完全用python开发的pdf解析库，对于线框完全的表格，pdfminer能给出比较好的抽取效果，但是对于线框不完全（包含无线框）的表格，其效果就差了不少。因为在实际项目所需处理的pdf文档中，线框完全及不完全的表格都比较多，所以为了能够理解pdfplumber实现表格抽取的原理和方法，找到改善、提升表格抽取的方法，这里对pdfplubmer的代码逻辑进行了梳理。由于所涉及的内容比较多，所以计划分为三部分进行整理：1. 介绍pdfplumber及其表格抽取流程, 2. 梳理pdfplumber表格线检测逻辑, 3. 梳理pdfplumber表格生成逻辑。本文是第一部分。
copyright: true
date: 2019-12-02 17:42:49
---


- [背景介绍](#%e8%83%8c%e6%99%af%e4%bb%8b%e7%bb%8d)
- [pdfplumber简介](#pdfplumber%e7%ae%80%e4%bb%8b)
- [pdfplumber抽取表格的基本流程](#pdfplumber%e6%8a%bd%e5%8f%96%e8%a1%a8%e6%a0%bc%e7%9a%84%e5%9f%ba%e6%9c%ac%e6%b5%81%e7%a8%8b)

# 背景介绍

最近在做一个表格信息抽取的项目，该项目需要从pdf文件中找到的目标表格，并把目标表格中需要的行和列给抽取出来。由于项目中pdf扫描件占比相对较少（不太到10%吧），所以目前主要把精力花在可编辑pdf文件的表格抽取上。

即便是可编辑的pdf文件，从中抽取表格也不是一件容易的事情，概括起来，难在以下几点：

1. 与其说pdf是一种数据格式，不如说它是一组打印指令的集合，因为pdf文件保存的只是一条条打印指令，这些指令告诉pdf阅读器或打印机该在屏幕或者纸张的什么位置显示什么样的符号。与docx和html等格式的文件不同（docx和html通过标签的方式组织不同的逻辑结构，比如<table>, <w:tbl>, <p>, <w:p>等），pdf文件不包含任何逻辑结构的信息，比如段落、句子、单词、表格等等。在pdf文档中，即便在阅读器中能看到`table-like`的东西，但是却无法直接有效地把这些视觉上`table-like`的东西所对应的数据给抽取出来。
2. 除了不会保存逻辑结构信息之外，pdf往往也不会保存空格、制表符、回车等不可见字符，所以在pdf中无法像在docx中一样，通过制表符来定位不是用线框表示的表格。

为了从pdf中比较好的抽取表格，作者调研、尝试了许多开源的框架（不限于python开发的框架），包括微软开源的深度学习表格检测与识别模型[TableBank](https://github.com/doc-analysis/TableBank)。尝试了一圈下来，在基于python的框架中，pdfplumber和camelot的效果相对较好。对于线框完全的表格，二者都能给出比较好的抽取效果，但是对于线框不完全（包含无线框）的表格，二者的效果就差了不少。

因为在项目所需处理的pdf文档中，线框完全及不完全的表格都比较多，所以为了能够理解pdfplumber实现表格抽取的原理和方法，找到改善、提升表格抽取的方法，作者在这里对pdfplubmer的代码逻辑进行了梳理。由于所涉及的内容比较多，所以计划分为三部分进行整理，分别是：

1. pdfplumber是怎么做表格抽取的（一）：介绍pdfplumber及其表格抽取流程
2. pdfplumber是怎么做表格抽取的（二）：梳理pdfplumber表格线检测逻辑
3. pdfplumber是怎么做表格抽取的（三）：梳理pdfplumber表格生成逻辑

本文是第一部分。


# pdfplumber简介

[pdfplumber](https://github.com/jsvine/pdfplumber)是一款基于[pdfminer](https://github.com/euske/pdfminer)，完全由python开发的pdf文档解析库，不仅可以获取每个字符、矩形框、线等对象的具体信息，而且还可以抽取文本和表格。目前pdfplumber**仅支持可编辑的pdf文档**。

虽然pdfminer也可以对可编辑的pdf文档进行解析，但是比较而言，pdfplumber有以下优势：

1. 二者都可以获取到每个字符、矩形框、线等对象的具体信息，但是pdfplumber在pdfminer的基础上进行了封装和处理，使得到的对象更易于使用，对用户更友好。
2. 二者都能对文本解析，但是pdfminer输出的文本在布局上可能与原文差别比较大，但是pdfplumber抽取出的文本与原文可以有更高的一致性。
3. pdfplumber实现了表格抽取逻辑，基于最基本的字符、线框等对象的位置信息，定位、识别pdf文档中的表格。


# pdfplumber抽取表格的基本流程

pdfplumber把表格抽取的功能封装在`TableFinder`这个类中，在其构造函数`__init__`中，清晰的定义了表格抽取的基本流程。下面截取了`TableFinder`类`__init__`函数部分的代码：

```python
class TableFinder(object):
    """
    Given a PDF page, find plausible table structures.

    Largely borrowed from Anssi Nurminen's master's thesis: http://dspace.cc.tut.fi/dpub/bitstream/handle/123456789/21520/Nurminen.pdf?sequence=3

    ... and inspired by Tabula: https://github.com/tabulapdf/tabula-extractor/issues/16
    """
    def __init__(self, page, settings={}):
        for k in settings.keys():
            if k not in DEFAULT_TABLE_SETTINGS:
                raise ValueError("Unrecognized table setting: '{0}'".format(
                    k
                ))
        self.page = page
        self.settings = dict(DEFAULT_TABLE_SETTINGS)
        self.settings.update(settings)
        for var, fallback in [
            ("text_x_tolerance", "text_tolerance"),
            ("text_y_tolerance", "text_tolerance"),
            ("intersection_x_tolerance", "intersection_tolerance"),
            ("intersection_y_tolerance", "intersection_tolerance"),
        ]:
            if self.settings[var] == None:
                self.settings.update({
                    var: self.settings[fallback]
                })
        self.edges = self.get_edges()
        self.intersections = edges_to_intersections(
            self.edges,
            self.settings["intersection_x_tolerance"],
            self.settings["intersection_y_tolerance"],
        )
        self.cells = intersections_to_cells(
            self.intersections
        )
        self.tables = [ Table(self.page, t)
            for t in cells_to_tables(self.cells) ]
```

pdfplumber抽取表格主要包含以下几步：

1. 因为表格以及单元格都是存在边界的（由可见或不可见的线表示），所以第一步，pdfplumber是找到可见的或猜测出不可见的候选表格线。
2. 因为表格以及单元格基本上都是定义在一块举行区域内，所以第二步，pdfplumber是根据候选的表格线确定它们的交点。
3. 根据得到的交点，找到它们围成的最小的单元格。
4. 把连通的单元格整合到一起，生成一个检测出的表格对象。

好了，这部分就初步写到这里吧 ^_^