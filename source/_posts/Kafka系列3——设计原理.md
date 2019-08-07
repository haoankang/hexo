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
> 一个kafka分区本质上是一个备份日志，利用多份相同备份共同提供冗余机制来保持系统高可用性，这些备份在kafka中被称为副本，kafka把分区所有副本均匀分配到
所有broker上，并从中挑选一个作为leader副本对外提供服务，其他被称为follower副本，只能被动向leader请求数据，保持与leader同步；ISR就是kafka集群动态
维护的一组同步副本集合，每个topic分区都有自己的ISR列表，leader副本总是包含在ISR中；<br>
> 1.3.1 follower副本同步————比较重要的几个位移信息：起始位移，高水印值（HW），日志末端位移（LEO）；HW指向消息，LEO指向下一个待写入消息<br>
> 1.3.2 ISR设计————ISR是一个动态列表，follower与leader不同步主要是如下原因：请求速度追不上，进程卡住，新创建的副本；老版本用消息数量位移来判断是否同步，
新版本用时间，参数replica.lag.time.max.ms，默认10s，检测机制：如果一个follower副本落后leader的时间持续性地超过这个参数值，那么判断为不同步；

1.4 水印和leader epoch
> 前面介绍了HW和LEO，HW是消费者可以读取到的消息位移，LEO可以理解为生产者生产的最新消息位移；<br>
> 1.4.1 LEO更新机制：kafka设计了两套follower副本LEO属性，一套LEO值保持在follower副本所在的broker缓存上，另一套保存在leader副本所在的broker缓存上；
>* follower副本端的follower副本LEO何时更新：follower副本端的LEO值就是其底层日志的LEO值，也就是说每当新写入一条消息，其LEO值就会被更新(类似于LEO += 1)。
当follower发送FETCH请求后，leader将数据返回给follower，此时follower开始向底层log写数据，从而自动地更新LEO值。
>* leader副本端的follower副本LEO何时更新：leader副本端的follower副本LEO更新发生在leader处理follower FETCH请求时，一旦leader收到请求，首先会从自己log
中读取相应数据，在给follower返回数据前会先比较FETCH参数更新follower的LEO；
>* leader副本更新leader的LEO：leader写log时会自动更新自己的LEO；

> 1.4.2 HW更新机制：
>* follower副本的HW更新：发生在更新LEO之后，具体算法是比较当前LEO值与FETCH响应中leader的HW值，取两者的小者作为follower的新HW值；
>* leader副本的HW更新：以下4种情况leader会尝试更新分区HW值————1.副本成为leader时；2.broker出现崩溃导致副本被踢出ISR时；3.producer向leader写入消息时，
4.leader处理follower FETCH请求时；如何更新？leader保存了分区所有的LEO，尝试确定分区HW时，会选出所有满足条件的副本（满足条件之一：处于ISR中，副本LEO落后于
leader LEO时长不大于replica.lag.time.max.ms），比较它们的LEO，选择最小的LEO作为HW值；

> 1.4.3 kafka备份原理
基于水印的更新原理是利用HW值来决定副本备份进度，而HW值更新需要另外一轮FETCH请求才能完成，一次数据写入需要两轮fetch完成同步；因此本质上有缺陷，可能引起备份数据丢失或者备份数据不一致；
![基于水印更新过程](/images/Kafka_8.png)                            
>因此在0.11.0.0版本中引入leader epoch值解决问题；leader端开辟一段内存区域专门保存leader的epoch信息，定时写入一个检查点文件中（持久化）；leader eopch实际上
是一对值(epoch,offset)，epoch表示leader的版本号，从0开始，offset相当于LEO；

1.5 日志存储设计
> kafka的日志设计都是以分区为单位，每个分区有自己的日志（分区日志）；每个分区日志都是由若干组日志段文件+索引文件组成；<br>
> 创建topic时，kafka为每个分区在文件系统中创建一个对应的子目录，名字是<topic>-<分区号>.kafka的每个日志段文件是由上限大小的，由broker参数log.segment.bytes控制，
默认1GB；日志段文件填满记录后，kafka会自动创建一组新的日志段文件和索引文件（日志切分）；kafka正在写入的分区日志段文件成为当前激活日志段或当前日志段.active log segment.
