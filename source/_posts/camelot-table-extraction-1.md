---
title: camelot是怎么做表格抽取的（一）—— camelot框架概览
tags:
  - 表格抽取
  - 表格检测
  - 表格识别
  - 开源框架
categories: 表格抽取
copyright: true
date: 2020-01-13 23:44:45
description:
---


- [背景介绍](#%e8%83%8c%e6%99%af%e4%bb%8b%e7%bb%8d)
- [camelot相关资源](#camelot%e7%9b%b8%e5%85%b3%e8%b5%84%e6%ba%90)
- [camelot功能](#camelot%e5%8a%9f%e8%83%bd)
  - [功能完备性](#%e5%8a%9f%e8%83%bd%e5%ae%8c%e5%a4%87%e6%80%a7)
  - [两种表格抽取模式](#%e4%b8%a4%e7%a7%8d%e8%a1%a8%e6%a0%bc%e6%8a%bd%e5%8f%96%e6%a8%a1%e5%bc%8f)

# 背景介绍

最近在做一个表格信息抽取的项目，该项目需要从pdf文件中找到的目标表格，并把目标表格中需要的行和列给抽取出来。由于项目中pdf扫描件占比相对较少（不太到10%吧），所以目前主要把精力花在可编辑pdf文件的表格抽取上。更多的背景介绍这里就不展开了，感兴趣的朋友可以到[冰焰虫子-github pages](https://iceflameworm.github.io/)或[探索发现频道-知乎专栏](https://www.zhihu.com/people/iceflameworm/columns)找《pdfplumber是怎么做表格抽取的》系列看一下。

camelot是作者调研的众多开源框架中效果相对比较好的之一，接下来，本文将对camelot框架进行简单的梳理，主要包括与camelot相关的一些资源以及camelot的各项功能。有关camelot具体功能的梳理与剖析会在后续的文章中陆续给出，欢迎各位看官阅读、点赞、收藏 ^_^。

# camelot相关资源

- [camelot项目页](https://camelot-py.readthedocs.io/en/master/)
- camelot代码仓库，有两个，其中一个可以直接通过上面的项目页跳转，可以把两个仓库看作是项目的不同分支。
  - [camelot-dev](https://github.com/camelot-dev/camelot): 这个可以直接从项目跳转，但是只有400多星，why？
  - [atlanhq](https://github.com/atlanhq/camelot)：这个2600星
- Excalibur:基于camelot的前端可视化pdf抽取工具，可以通过Excalibur体验、测试camelot的效果。
  - [Excaliblur项目首页](https://excalibur-py.readthedocs.io/en/master/)
  - [Excaliblur github](https://github.com/camelot-dev/excalibur)
- camelot实现了Anssi Nurminen硕士论文中用于抽取非线框类表格的算法
  - [Anssi Nurminen’s master’s thesis](https://dspace.cc.tut.fi/dpub/bitstream/handle/123456789/21520/Nurminen.pdf)

# camelot功能

## 功能完备性

camelot是一个可以从可编辑的pdf文档中抽取表格的开源框架，与pdfplumber相比，其功能的完备性要差不少，因为除了表格抽取之外，并不能用它从pdf文档中解析出字符、单词、文本、线等较为低层次的对象。在表格抽取的过程中，camelot使用pdfminer实现底层对象的解析，但是这些底层对象的抽取逻辑并没有封装成通用的函数或方法，所以用camelot获取底层对象还是不太方便的。

## 两种表格抽取模式

camelot的主要功能是表格抽取，支持`lattice`和`stream`两种不同的模式，其中`lattice`用来抽取线框类的表格，`stream`用来抽取非线框类的表格。

在抽取线框类表格的时候，`lattice`包含以下几步：

1. 把pdf页面转换成图像
2. 通过图像处理的方式，从页面中检测出水平方向和竖直方向可能用于构成表格的直线。
3. 根据检测出的直线，生成可能表格的bounding box
4. 确定表格各行、列的区域
5. 根据各行、列的区域，水平、竖直方向的表格线以及页面文本内容，解析出表格结构，填充单元格内容，最终形成表格对象。

`stream`包含以下几步：

1. 通过pdfminer获取连续字符串
2. 通过文本对齐的方式确定可能表格的bounding box
3. 确定表格各行、列的区域
4. 根据各行、列的区域以及页面上的文本字符串，解析表格结构，填充单元格内容，最终形成表格对象。