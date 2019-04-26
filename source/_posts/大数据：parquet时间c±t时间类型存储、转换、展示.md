---
title: 大数据：parquet时间类型存储、转换、展示
date: 2019-04-18 14:21:07
categories: big data
tags: 大数据
---

## parquet时间类型存储、转换、读取
1. hive表转parquet. (10.100.2.100:50070)
    /user/hive/warehouse/csv2parquet
2. java读取parquet并解析时间类型.

3. spark写parquet文件格式.

4. 解析上面的时间类型.

## 部分结论
1. spark存储可以设置时间类型是long类型，然后正常转换得到；
2. hive存储在parquet中的时间类型是int96，可以通过解析得到；

## 详细摘要
1. hive创建parquet文件格式的数据.
    a. create table haoaktest(
            id int,
            mytime timestamp,
            mydate date
        ) row format delimited fields terminated by ',';
        
    b. load data local inpath '/home/hadoop/haoak/haoaktest.csv' into table haoaktest;
        注意hive的timestamp类型是"yyyy-MM-dd HH:mm:ss.SSS", date类型是"yyyy-MM-dd".
        
    c. create table csv2parquet stored as parquet
        as select * from haoaktest;
        
    d. show create table csv2parquet可以看到路径和文件格式，注意文件路径是hive在hdfs上的默认目录下；