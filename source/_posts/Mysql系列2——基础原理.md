---
title: '''Mysql系列2——基础原理'''
date: 2019-12-24 14:37:12
categories: mysql
tags: mysql
---
**1.Mysql客户端和服务端通信方式**

>* TCP/IP通信，最常用方式，例如客户端连接：msql -hhost -uusername -ppasswd -P3306 [database]
>* 命令管道和共享内存，windows可选连接方式.
>* Unix域套接字文件，类unix系统可选.

**2.服务端处理客户端请求**

![](/images/Mysql_1.png)
整个过程分为三个部分：连接管理、解析优化、存储引擎.
常用命令：
```sql
show engines;
create table tableName() engine=InnoDB;
alter table tableName engine=MyISAM;
```

**3.服务端参数**

服务端启动参数可以直接在启动时添加，例如mysqld --skip-networking；

服务端启动时读取配置文件my.ini或my.cnf，按照一定顺序读取；

配置文件中，按照分组概念配置，不同启动命令可以读取不同的组；

服务器启动后，有很多系统变量，影响着服务器的行为，显示：show variables [like (匹配项)];

同时服务器启动后，会有状态变量，可以查看服务器的状态：show status [like (匹配项)];

**4.字符集和比较规则**

mysql中UTF-8分为utf8和utf8mb4，utf8是阉割版的utf8，1-3个字节，utf8mb4对应utf8，1-4个字节；
```sql
show character set/charset (like [匹配项]);
show collation (like [匹配项]);
```
mysql中比较规则后缀：_ai和_as区分重音，_ci和_cs区分大小写，_bin以二进制形式比较；

mysql中有4个字符集和比较规则级别：服务器、数据库、表、列；
```sql
编解码过程：
character_set_client: 服务器解码请求时使用字符集
character_set_connection: 服务器处理请求时会把请求字符串从character_set_client转码到character_set_connection;
character_set_result: 服务器向客户端返回结果时使用的字符集
一般建议这三个参数设置一样.
```