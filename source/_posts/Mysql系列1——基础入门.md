---
title: Mysql系列1——基础入门
date: 2019-11-25 16:06:48
categories: mysql
tags: mysql
---

##1. mysql的安装、启动和命令行.
>Mysql的安装可以官网下载相应版本，windows可以直接下载免安装解压包，配置环境变量；
也可以linux安装或者docker，这些不是本文重点，不再累赘说明；

>通过mysql客户端工具访问，语法如下：
>>* mysql -h127.0.0.1 -uroot -p123456 -P3306 my-database，这里建议参数间不加空格，
因为-p后接密码时不能输入空格，可以直接指定数据库my-database，注意用户必须指定，不然默认
用户ODBC;
>>* 查看mysql的版本，可以根据情况使用status(mysql命令行)，或者select version();
>>* 命令行操作时，\G格式化输出,\c取消当前命令操作；可以一次批量操作；

##2. mysql的数据类型.
>* 整数类型：tinyInt(1字节)、smallInt(2)、mediumInt(3)、int(4)[别名integer]、bigInt(8).
>* 浮点数类型：float(4字节)、double(8字节)，对于float(M,N)，表示一共有M个位数，N个小数，
计算机是用二进制表示的，
9.875=8+1+0.5+0.25+0.125=1*2³+0*2²+0*2¹+1*2⁰+1*2ˉ¹+1*2ˉ²+1*2ˉ³，即1001.111，
这种情况0.3这种就没法精确表示，因此浮点数类型是不精确的；
>* 定点数类型：decimal(M,N)，定点数类型取值和占用字节不固定，思路是对一个有小数的数字而言，
可以拆分为整数部分和小数部分，这两部分都用整型表示，即1.3= 1和3；
>* 无符合数值类型表示：UNSIGNED
>* 日期和时间类型: year(1字节)、date(3字节)、time(3字节)、datetime(8字节)、timestamp(4字节)；
datetime(M)表示精确到M位毫秒，建议用timestamp，可以根据设置的时区显示不同的时间，而且占用空间小，
本质是一个int类型；
>* 字符串类型：char、varchar、tinytext、text、mediumtext、longtext；
>* enum类型和set类型；
>* 二进制类型：bit、binary、varbinary、tinyblob、blob、mediumblob、longblob；

##3. 数据库的基本操作.
>* create datable (if not exists) xxx;
>* drop database (if not exists) xxx;
>* show databases;
>* use xxx;

##4. 表的基本操作.
>* show tables;
>* create table (if not exists) xxx(schema1 int primary key, schema2 varchar(10));
>* drop table (if exists) xxx;
>* desc database.xxx;
>* show create table database.xxx;
>* alter table old_name rename to new_name;/rename old_name to new_name;（也可以用于转移数据库）
>* alter table table_name add column schema_name schema_type [schema_param] [first/after schema_name];
>* alter table table_name drop column schema_name;
>* alter table table_name modify schema_name schema_type [schema_param] [first/after schema_name];;

##5. 列的属性.
> 常用的列的属性主要有：null、key(primary key、unique、foreign key)、default、extra(例如auto_increment等);
> primary key和unique key的区别是：
>> * 一个表的primary key只能有一个，unique key可以多个；
>> * primary key修饰的列不能为null，unique修饰的列可以；

##6. 简单查询.
```sql
select [distinct] 查询列表
[from 表名]
[where 布尔表达式]
[group by 分组列表]
[having 分组过滤条件]
[order by 排序列表]
[limit 开始行、限制条数]
```