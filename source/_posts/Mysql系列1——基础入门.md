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
>注意查询的顺序，聚集函数一般配合分组使用，且聚集函数不能用在where子句中；

##7. 子查询
>* 简单样式：select * from a where a.name=(select b.name from b);
>* 子查询括号内的是内层查询，括号外的是外层查询；
>* 根据查询结果分为：标量子查询、列子查询、行子查询、表子查询；
>* 其他的概念：exists子查询和not exists子查询，不相干子查询和相关子查询；

##8. 连接查询
>* 连接查询可以查询N张表，连接产生的笛卡儿积数量中间数据；
>* 连接分为内连接和外连接，区别是：驱动表中的记录即使在被驱动表中没有匹配记录，是否需要
加入到结果集；需要是外连接，不需要是内连接；
>* 内连接两种形式：select * from a,b where a.id=b.id;或者:
select * from a inner join b on a.id=b.id;由于内连接和where子句是等价的，所以
内连接不需要强制使用on子句；
>* 外连接分为左连接left join和右连接right join.
>* 对于内连接来说，驱动表和被驱动表是可互换的；对外连接来说不可互换；
>* 如果一个表关联自己查询，被称为自查询；

##9. 组合查询
> union和union all查询区别是：union会自动去重，union all会保留重复行.
组合查询中可以使用group by和limit.

##10. 增删改的一些细节
>* insert ignore————对于哪些主键或UNIQUE约束的列或列组合来说，如果表中已存在记录的列中没有与待插入
记录在这些列或列组合上重复的值，则插入，否则忽略;
>* insert on duplicate key update————同上，如果已存在记录，则更新；

##11. 视图
>* 语法： create view 视图名 as 查询语句
>* 视图本质是查询语句的别名，不会把结果存储，当查询视图时，会转换成相应的查询语句进行
查询操作；例如当更新表后，查询视图结果也会改变；
>* 一般视图只在查询语句中使用，增删改时最好不要使用视图；

##12. 存储程序
>* 存储程序可以封装一些语句，提供一种简单方式调用存储程序，相当于这些语句的别名；
存储程序包括存储例程（存储过程和存储函数）、触发器、事件；
>* mysql通过@定义变量，例如@a、@b等；通过delimiter定义语句结束分隔符.
>* 存储函数.
>>* 本质是一个存储sql语句的函数;
>>* 定义语法：注意如果需要使用;可以提前定义delimiter;
```sql
create function 存储函数名称(【参数列表】)
returns 返回值类型
begin 
    函数主体
end 
```
>>* 使用方式：和内置行数如avg(),count()等一样使用；
>>* 在函数体中定义变量格式： declare 变量1,变量2, ... 数据类型 [default 默认值];
>>* 判断语句格式
```sql
if 表达式 then 
    处理语句列表
[elseif 表达式 then 
    处理语句列表]
    ...
 [else 
    处理语句列表]
end if;
```
>>* 循环语句格式
```sql
while 表达式 do
    处理语句列表
end while;

repeat 
    处理语句列表
until 表达式
end repeat;

loop 
    处理语句列表
end loop;

```
>* 存储过程
>>* 存储过程和存储函数都属于存储例程；都是对某些语句的封装，存储函数侧重于执行函数
并返回一个值，存储过程侧重于单纯的执行；