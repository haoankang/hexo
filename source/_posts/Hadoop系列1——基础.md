---
title: '''Hadoop系列1——基础'''
date: 2020-03-02 15:34:13
categories: hadoop,大数据
tags: hadoop,大数据
---
###1. HDFS架构.
>HDFS采用Master/Slave架构，一个HDFS集群包含一个单独的NameNode节点和多个DataNode节点.
>
>NameNode负责管理整个分布式系统的元数据，主要包括：目录树结构、文件到数据库Block的映射关系、Block副本及其
>存储位置等数据管理、DataNode的状态监控；这些数据保存在内存中，同时在磁盘保存两个元数据管理文件：fsimage和
>editlog；
>>* fsimage：是内存命名空间元数据在外存的镜像文件；
>>* editlog：各种元数据操作的write-ahead-log文件，在体现到内存数据变化前首先会将操作记录editlog中，以防止
>数据丢失；这两个文件结合可以构建完整的内存数据；
>
>Secondary NameNode.定期从NameNode拉取fsimage和editlog文件，并对两个文件进行合并，形成新的fsimage文件并
>传回NameNode，目的是减轻NameNode的压力，本质上是一个提供检查点功能服务的服务点；
>
>DataNode负责数据块的实际存储和读写工作，Block默认是64MB，当客户端上传一个大文件时，HDFS会自动将其切割成固定
>大小的Block，为了保证数据可用性，每个Block会以多备份的形式存储，默认是3份；

###2. MapReduce过程.
>MapReduce分为两个阶段：Map和Reduce.
>
>Map阶段：
>1. input.在进行map计算之前，mapreduce会根据输入文件计算输入分片，每个输入分片针对一个map任务；
>2. map.就是程序员编写好的map函数了，因此map函数效率较好控制，而且一般map操作都是本地化操作也就是在数据存储
>节点上进行；
>3. Partition.需要计算每个map结果需要发送到哪个reduce端，partition数等于reduce数，默认采用HashPartition；
>4. spill.此阶段分为sort和combine，首先分区过的数据会经过排序之后写入环形内存缓冲区，在达到阈值后守护线程
>将数据溢出分区文件；
>5. merge.spill结果会有很多个文件，merge会合并所有本地文件，并且该文件会有一个对应的索引文件；
>
>Reduce阶段：
>1. copy.拉取数据，reduce启动数据copy线程，通过http请求对应节点的map task输出文件，copy的数据也会先放到
>内部缓冲区，之后再溢写，类似map操作；
>2. merge.合并多个copy的多个map端的数据，在一个reduce端先将多个map端的数据溢写到本地磁盘，之后再将多个文件
>合并成一个文件，数据经过内存-》磁盘-》磁盘的过程；
>3. output.merge阶段最后会生成一个文件，并将此文件转移到内存中，shuffle阶段结束；
>4. reduce.开始执行reduce任务，最后结果保留在hdfs上；

