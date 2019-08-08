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
默认1GB；日志段文件填满记录后，kafka会自动创建一组新的日志段文件和索引文件（日志切分）；kafka正在写入的分区日志段文件成为当前激活日志段或当前日志段.active log segment.<br>
> 每个分区除了日志文件，还有位移索引文件和时间戳索引文件，按照规律升序排列，属于稀疏索引，利用升序规律，kafka可以二分查找搜寻索引，时间复杂度O(lgN).索引保存的是相对位移，
索引文件大小设置参数log.index.size.max.bytes；索引项密度参数log.index.interval.bytes.<br>
> 日志清理：两种留存策略——基于时间和基于大小，日志清理对当前日志段不生效；<br>
> 日志compaction. 针对topic设置，提供更细粒度化的留存策略，本质是针对K-V的消息，删除之前的消息，只保留最新的value消息；为了实现log compaction,kafka在逻辑上将
每个log分成log tail和log head，log head连续递增，log tail位移不连续；kafka的组件cleaner负责执行compaction；

1.6 通信协议
> Kafka的通信协议是基于TCP之上的二进制协议，这套协议所有类型的请求和响应都是结构化的，由不同的初始类型构成；broker端可配置参数用于限制broker端能处理请求的最大字节数，
超过阈值请求的socket连接会被强制关闭；kafka通信协议中规定的请求发送流向由3种——clients给broker发送请求、controller给其他broker发送请求、follow副本所在broker向leader
副本所在broker发送FETCH请求；clients与broker传输数据时，需要创建一个连向特定broker的socket长连接，单个clients通常需要连接多个broker，每个broker只需要维护一个socket；
kafka自带的java clients使用类似于epoll方式在当个连接上不停轮询传输数据；broker端需要确保在单个socket连接上按照发送顺序处理请求；<br>
> 请求/响应结构。kafka协议提供的所有请求及响应结构体都由固定格式组成，统一构建于多种初始类型之上，初始类型由：固定长度初始类型（int8,int16,int32,int64），可变长度初始
类型（bytes和string），数组；所有请求和响应统一格式——Size+Request/Response.
>>* 请求分为请求头部和请求体，请求头结构固定，由4个字段组成——api_key(int16)、api_version(int16)、correlation_id(int32)、client_id(string)；
>>* 响应分为响应头部和响应体，响应头结构固定，1个字段——correlation_id，与请求头中字段对应；
>>* 常见请求类型：PRODUCE请求(api_key=0)，V6版本的produce请求格式为——事务ID+acks+timeout+[topic数据]，响应结构——[response]+throttle_time_ms；FETCH请求(api_key=1)，
包括clients向broker发送的FETCH请求和follower给leader发送的FETCH请求，最新版本格式：replica_id+max_wait_time+min_bytes+max_bytes+isolation_level+[topics]，响应格式
throttle_time_ms[response]；METADATA请求，用于获取指定topic的元数据信息，格式[topics]+allow_auto_topic_creation，响应格式多变；
> 请求处理流程.![clients端](/images/Kafka_9.png)  ![broker端](/images/Kafka_10.png)

1.7 controller设计
> kafka集群中，某个broker会被选举为controller，用来管理和协调kafka集群，具体是管理集群中所有分区状态并执行相应管理操作。![](/images/Kafka_11.png)
> controller管理状态，维护的状态分为两类：每台broker上的分区副本和每个分区的leader副本信息，从维度上分为副本状态和分区状态；![](/images/Kafka_12.png)
> controller职责：更新集群元数据信息、创建topic、删除topic、分区重分配、preferred leader副本选举、topic分区扩展、broker加入集群、broker崩溃、受控关闭、controller leader选举；

1.8 broker请求处理
> broker处理请求模式是Reactor设计模式；![](/images/Kafka_13.png)
> processor线程数量可通过num.network.threads配置，broker会为用户配置的每组listener创建一组processor线程；processor线程一个重要任务是将socket连接上接收到的请求放入请求队列中，
KafkaRequestHandler线程池专门处理请求；请求队列由参数queued.max.requests控制，默认500，如果发送请求超过，发送给这个broker的请求会被阻塞；broker请求处理类似于主从Reactor多线程
模型；

##2 producer端设计
2.1 producer基本数据结构.
>producer端与broker的主要交互是发哦是那个信息然后接收回调请求；因此如第2节的代码示例，主要数据结构是ProducerRecord和RecordMetadata.

2.2 工作流程.
![](/images/Kafka_14.png)
详细流程如下：
>* 序列化+计算目标分区；
>* 追加写入消息缓冲区；producer创建时会创建一个默认32MB的缓冲区，用于保存待发送的消息；关键参数有linger.ms、batch.size和消息批次信息(batches)，batches本质上是
一个HashMap，分别保存了每个topic分区下的batch队列，例如{"test-0"->[batch1,batch2],"test-1"->[batch3]}；
>* sender线程预处理及消息发送；
>* sender线程处理response；

##3 consumer端设计
3.1 consumer group状态机. ![](/images/Kafka_15.png)

3.2 group管理协议. 
> coordinator的组管理协议由两个阶段构成——组成员加入阶段和状态同步阶段。第一个阶段用于为group指定active成员并从中选出leader consumer，第二个阶段让leader consumer
制定分配方案并同步到其他组成员中；

##4 实现精确一次处理语义
4.1 消息交付语义：最多一次，最少一次，精确一次；

4.2 kafka如何实现精确一次处理语义：幂等性producer（PID，producer自行分配）和事务（TransactionalId，用户显示提供）

本文大部分内容摘自《Apache Kafka实战》