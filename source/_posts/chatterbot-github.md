---
title: 开源框架gunthercox/ChatterBot浅析
date: 2019-08-12 16:57:13
tags:
    - NLP
    - chatbot
    - 开源框架
categories: chatbot
description: 简单梳理一下github上开源的一个聊天机器人框架：gunthercox/ChatterBot，内容主要包含工作流程，训练器和逻辑适配器三部分。由于作者初涉chatbot这一领域，所以对某些部分的理解可能不到位，敬请谅解。
---

转载请注明来源：http://iceflameworm.github.io/2019/08/12/chatterbot-github/

# 框架简介
Chatterbot是一个完全用python编写的基于文本检索/匹配的聊天机器人框架，它会从保存的对话语料中找出与输入句子最匹配的句子，并把匹配到的句子的下一句作为回答返回。

框架地址：https://github.com/gunthercox/ChatterBot   
文档地址：https://chatterbot.readthedocs.io/en/stable/

# 工作流程
![img](workflow.png)

原文档中有两幅描述工作流程的示意图，一幅在[文档首页](https://chatterbot.readthedocs.io/en/stable/index.html)，一幅在[文档-逻辑适配器](https://chatterbot.readthedocs.io/en/stable/logic/index.html)，个人认为后者描述的更全面、更恰当些，所以就以后者为准进行介绍。

从输入句子到输出响应回答，前后需要经历三大步：
1. 预处理
2. 生成答案
3. 答案选择

## 预处理
与通常所讲的NLP预处理的目的基本一致，主要是文本进行一些标准还操作，比如去除连续的空格、删除特殊字符等等。Chatterbot框架自身实现了`clean_whitespace`, `unescape_html`和`convert_to_ascii`三种预处理功能。具体实现参见[preprocessors.py](https://github.com/gunthercox/ChatterBot/blob/master/chatterbot/preprocessors.py)

## 生成答案

## 答案选择

# 训练器

# 逻辑适配器