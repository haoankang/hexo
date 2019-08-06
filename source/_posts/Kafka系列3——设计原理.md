---
title: Kafka系列3——设计原理
date: 2019-08-06 10:48:03
categories: kafka
tags: kafka
---

# Kafka设计原理
##1 broker端设计架构
1.1 消息设计
> kafka经历过三个消息格式版本，V0、V1、V2；目前新版本用的是V2格式；<br>
> kafka消息集合和消息层次的概念：无论哪个版本kafka，消息层次分为两层：消息集合和消息，一个消息集合包含若干个日志项，每个日志项都封装了实际的消息和一组元数据信息，
kafka日志文件就是由一系列消息集合日志项构成。kafka不会在消息层面上直接操作，总是在消息集合上进行写入操作；<br>
> V2之前版本使用的是日志项，V2版本使用消息批次，可以理解为同一个东西。![](/images/Kafka_3.png) ![](/images/Kafka_4.png)
> V0、V1、V2消息格式对比：![](/images/Kafka_5.png) ![](/images/Kafka_6.png) ![](/images/Kafka_7.png)
> 综上，V2版本格式比之前复杂许多，单个消息占用磁盘空间也增加了；但考虑到使用场景，大数据量时V2版本优秀许多；

1.2 集群管理
> kafka依赖zookeeper实现分布式的消息引擎集群环境，支持自动化的服务发现和成员管理。<br>
> 理解zookeeper路径有助于理解kafka:
>* /brokers: 保存了kafka集群的所有信息，包括每台broker的注册信息，集群上所有topic信息等；
>* /controller: 保存了kafka controller组件的注册信息，同时也负责controller的动态选举；
>* /admin: 保存管理脚本的输出结果，比如删除topic，对分区进行重分配等操作；
>* /isr_change_notification: 保存ISR列表发生变化的分区列表。controller会注册一个监听器实时监控该节点下子节点的变更；
>* /config: 保存了kafka集群下各种资源的定制化配置信息；
>* /cluster: 保存kafka集群的简要信息；
>* /controller_epoch: 保存controller组件的版本号，用来隔离无效的controller请求；

1.3 副本与ISR设计
> 