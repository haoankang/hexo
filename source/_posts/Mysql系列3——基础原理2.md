---
title: '''Mysql系列3——基础原理2'''
date: 2019-12-25 20:38:12
categories: mysql
tags: mysql
---

**1.Mysql数据目录**
数据终究要存储在磁盘上，因此Mysql数据目录中包含各种文件，例如表空间、表、视图、日志、进程文件等；因此数据库受文件系统的影响：
>* 数据库名称和表名称不得超过文件系统所允许的最大长度
>* 特殊字符问题
>* 文件长度受文件系统最大长度限制

mysql中有几个系统数据库：
>* mysql: 存储了Mysql的用户账户和权限信息，一些存储过程、事件的定义信息，一些日志信息、帮助信息以及时区信息；
>* information_schema: 维护所有其他数据库的信息，是一些描述性信息；
>* performance_schema: mysql服务器运行过程中的一些状态信息；
>* sys: 通过视图形式把information_schema和performance_schema结合起来，可以更方便的了解mysql服务器的一些性能信息；
>
