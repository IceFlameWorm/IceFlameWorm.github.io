---
title: 【严选】MongoDB及其在python和flask中的使用
copyright: true
date: 2020-02-07 21:49:14
tags:
    - MongoDB
    - Python
    - Flask
categories: 数据库
description: 近期的项目需求需要开发一个后台服务，该后台服务需要把中间状态和结果写到数据库中以便后续查询。出于开发的便捷性，最终选定了MongoDB。由于后台服务用python开发，服务框架基于flask，所以就搜集整理了MongoDB，及其在python和flask中使用的相关的资料。在这里把相关的资料记录分享出来，方便自己以及其它有需求的小伙伴查阅 ^_^。
---

# MongoDB可视化管理工具

## Roto 3T（原robomongo，以前用过还不错）

官网：https://robomongo.org/

Roto 3T包含：
1. Studio 3T（付费，免费试用30天）
2. Robo 3T（免费，更轻量级，类似于原有的robomongo）


# Mongodb的python客户端

## pymongo

**语法类似于mongo原生的语法**   
github - pymongo：https://github.com/mongodb/mongo-python-driver   
pymongo文档：https://pymongo.readthedocs.io/   
pymongo本身是**线程安全的，但是进程不安全**：https://pymongo.readthedocs.io/en/stable/faq.html#id1

## mongoengine

本身依赖pymongo   
**可以按照类似关系型数据库来定义数据的结构，使得操作风格和关系型数据库相似，可以使得代码更好的维持MVC结构。**   
官网：http://mongoengine.org/   
文档：http://docs.mongoengine.org/  
github：https://github.com/MongoEngine/mongoengine


## flask-pymongo

flask操作mongodb的插件，基于pymongo，**用法跟pymongo基本一致**   
flask-pymongo -  github：https://github.com/dcrosta/flask-pymongo   
flask-pymongo文档：https://flask-pymongo.readthedocs.io/

Flask 扩展 Flask-PyMongo：https://www.cnblogs.com/Erick-L/p/7047064.html


## flask-mongoengine

flask操作mongodb的插件，基于mongoengine   
flask-mongoengine - github：https://github.com/MongoEngine/flask-mongoengine   
flask-mongoengine文档：https://flask-mongoengine.readthedocs.io

flask mongoengine的使用(一)查询：https://www.jianshu.com/p/fe6727f549d2


# 参考资料

1. MongoDB官网：https://www.mongodb.com/
2. MongoDB中文站（感觉东西不是特别全，不如菜鸟和w3cschool）：https://www.mongodb.org.cn/
3. MongoDB学习笔记(六)——MongoDB配置用户账号与访问控制：https://blog.csdn.net/qq_33206732/article/details/79877948
4. MongoDB安装及设置服务启动：https://blog.csdn.net/dandanfengyun/article/details/95497728
5. MongoDB之python简单交互(三) - **pymongo, flask-pymongo, flask-mongoengine三者使用对比简介**：https://www.cnblogs.com/cwp-bg/p/9473144.html