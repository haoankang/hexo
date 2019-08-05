---
title: Kafka系列1——基础入门
date: 2019-08-01 17:23:53
categories: kafka
tags: kafka
---

Kafka的核心功能：高性能的消息发送和高性能的消息消费；
##1. 快速入门.
1.1 官网下载压缩包，例如kafka_2.12-2.3.0.tgz，其中2.12代表scala版本，2.3.0代表kafka版本.解压;
1.2 启动服务器：kafka用zookeeper做分布式协调服务，因此需要先启动zookeeper，使用解压包bin目录下
对应平台的命令，例如windows: 
```
  .\bin\windows\zookeeper-server-start.bat .\config\zookeeper.properties
```
然后另开终端启动kafka服务器，例如windows：
```
  .\bin\windows\kafka-server-start.bat .\config\server.properties
```
1.3 创建topic：kafka的topic用于消息的发送和接收，另开一个终端，执行命令：
```
  .\bin\windows\kafka-topics.bat --create --zookeeper localhost:2181 --topic test --partitions 1 --replication-factor 1
```
查看topic状态：
```
  .\bin\windows\kafka-topics.bat --describe --zookeeper localhost:2181 --topic test
```
1.4 发送消息和接收消息.
<br>另开两个终端，分别用于发送和接收消息，便于观察：
```
  producer: 
    .\bin\windows\kafka-console-producer.bat --broker-list localhost:9092 --topic test
  consumer:
    .\bin\windows\kafka-console-consumer.bat --bootstrap-server localhost:9092 --topic test --from-beginning
```
##2. 概述.
2.1 kafka是一个消息引擎，消息引擎设计时要考虑两大要素：消息设计（xml、json、二进制..)
和传输协议设计（AMQP、Webservice+SOAP、MSMQ..)；消息引擎范型：是一个基于网络的架构范型，
描述了消息引擎系统的两个不同的子部分是如何互联和交互的；常见的消息引擎范型有：消息队列模型和
发布/订阅模型.两者区别是同一条消息是否会被多个消费者消费；kafka同时支持两种范型，利用group的概念.

2.2 Java消息服务，Java Message Service(JMS)，是一套API规范，提供了很多接口用于实现分布式
系统间的消息传递，JMS同时支持两种消息引擎模型；目前主流的完全支持JMS规范的消息引擎有：ActiveMQ、
RabbitMQ、Kafka等.

2.3 Kafka设计解决的4个方面问题(特点).
> 2.3.1 吞吐量/延迟. kafka是如何做到高吞吐量和低延迟的？<br>
  a. kafka的写入操作很快。kafka持久化所有数据到磁盘，每次写入只是把数据写入到操作系统的
页缓存中（页缓存是内存中分配的，写入很快；kafka不必与底层文件系统打交道），且写入操作采用
追加写入(append)的方式，避免了磁盘随机写操作.因此kafka只能在日志末尾追加写入新消息，且不允许
修改已写入的消息. <br>
  b. kafka的消费端在读取消息时，首先会从OS的页缓存中读取，如果命中则直接把消息经页缓存发送到
网络的socket上。（这个过程利用linux平台的sendfile系统调用做到的，就是零拷贝，Java的FileChannel.transferTo方法实现）
而且由于大量使用页缓存，所以读取消息很大可能直接命中缓存，不用“穿透”到底层磁盘获取消息，从而极大提升吞吐量.<br>
  c. 总结.
>> * 大量使用操作系统页缓存，内存操作速度快且命中率高.
>> * kafka不直接参与物理I/O操作，交由操作系统完成.
>> * 采用追加写入方式，避免了缓慢的随机读写操作.
>> * 使用以sendfile为代表的零拷贝技术加强网络间的数据传输效率.
>
> 2.3.2 消息持久化.<br>
 解耦消息发送和消息消费；实现灵活的消息处理.
>
> 2.3.3 负载均衡和故障转移.<br>
 kafka通过zookeeper做分布式协调服务实现相应功能.
>
> 2.3.4 伸缩性.<br>
 kafka服务器上状态统一由Zookeeper保管，扩展很方便.

2.4 kafka的基本术语和概念.
>2.4.1 kafka的核心结构：生产者发送消息给kafka服务器；消费者从kafka服务器读取消息；kafka服务器依托zookeeper集群进行
服务的协调管理.
>
>2.4.2 消息.目前消息格式由三个版本，定义了消息的格式.
>2.4.3 topic和partition. topic是一个逻辑概念，代表一类消息，通过它实现两种消息引擎模型；kafka采用了topic-partition-message
的三级结构来分散负载.partition也没有太多业务含义，只是单纯为了提升吞吐量.<br>
>2.4.4 offset. kafka的消息是一个三元组<topic,partition,offset>.<br>
>2.4.5 replica. 实现高可靠性依靠冗余机制，就是副本（replica）.<br>
>2.4.6 leader和follower. 和传统主备不同，只有leader对外提供服务，follower只是被动追随leader状态，保持与leader
同步，存在的唯一价值是leader的候补.
>2.4.7 ISR. in-sync replica.与leader replica保持同步的replica集合.kafka承诺只要ISR集合中至少存在一个replica，
那些“已提交”状态的消息就不会丢失.

2.5 kafka的典型应用场景.
* 消息传输.
* 网站行为日志追踪.
* 审计数据收集.
* 日志收集.
* Event Sourcing. 使用事件序列来表示状态变更.
* 流式处理.
##3. 线上环境规划.
3.1 集群环境规划.
>3.1.1 操作系统选型.<br>
 单论与kafka的相适性，Linux比windows系统更适合部署.主要两个原因：I/O模型的使用和数据网络传输效率；kafka的clients底层网络库采用了Java
的Selector机制，这种机制在windows平台上使用select模型实现，在linux平台上使用epoll模型实现，通常epoll比select模型更好。
>
>3.1.2 磁盘规划.<br>
 因为kafka的写入机制是顺序IO写，因此SSD和普通磁盘性能差距不大；RAID比JBOD的优势在于天然的负载均衡和冗余机制，
 但这在kafka框架层面已经提供；因此性价比方面JBOD(一堆普通磁盘)更好.
>
>3.1.3 磁盘容量规划.<br>
 磁盘容量规划和多个因素有关：新增消息数、消息留存事件、平均消息大小、副本数、是否启用压缩.
>
>3.1.4 内存规划.<br>
 建议：尽量分配更多内存给操作系统的页缓存；不要为broker设置过大的堆内存；页缓存大小至少要大于一个日志段的大小。
>3.1.5 CPU规划.<br>
 建议使用多核心系统，CPU核心数最好大于8；如果启用消息压缩，需要更多cpu资源。
>3.1.6 带宽规划.<br>
 假设用户网络带宽1Gb/s，业务目标是每天用1小时处理1TB业务消息。分析：单台broker的带宽资源假设可以达到70%，则实际带宽1Gb/s*0.7=710Mb/s，
为了预防突发流量，再截取1/3，则710Mb/s/3=240Mb/s，因此单台broker的带宽估算为240Mb/s；1小时处理1TB业务消息，即每秒需要处理292MB左右的数据，也就是每秒2336Mb数据，
至少需要2336/240=10台broker机器，如果副本数2，则需要20台broker机器.
 
 