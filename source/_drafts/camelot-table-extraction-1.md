---
title: camelot是怎么做表格抽取的（一）—— camelot框架概览
copyright: true
date: 2020-01-03 23:44:45
tags:
    - 表格抽取
    - 表格检测
    - 表格识别
    - 开源框架
categories: 表格抽取
description:
---

- [背景介绍](#%e8%83%8c%e6%99%af%e4%bb%8b%e7%bb%8d)
- [camelot相关资源](#camelot%e7%9b%b8%e5%85%b3%e8%b5%84%e6%ba%90)
- [camelot功能](#camelot%e5%8a%9f%e8%83%bd)

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