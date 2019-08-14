---
title: Zookeeper系列1——概述
date: 2019-08-14 14:36:44
categories: zookeeper
tags: zookeeper
---

##1. zookeeper概述.
1.1 zookeeper是什么？
>zookeeper是一种用于分布式应用程序的开源分布式协调服务，是一个典型的分布式数据一致性解决方案，分布式应用可以基于zookeeper实现如数据发布/订阅、负载均衡、
命名服务、分布式协调/通知、集群管理、master选举、分布式锁和分布式队列等功能；<br>
zookeeper可以保证如下分布式一致性特性：顺序一致性、原子性、单一视图、可靠性、实时性；

1.2 zookeeper设计目标？
>* 简单的数据模型
>* 可以构建集群
>* 顺序访问
>* 高性能

1.3 zookeeper的数据模型和分层命名空间.
>zookeeper提供的命名空间类似于标准文件系统，名称由/分割的路径元素序列，命名空间中的每个节点都由路径标识。![](/images/zookeeper_1.png)

1.4 zookeeper的概念.
>* 集群角色————通常分布式系统中，典型的集群模式是Master/Slave模式（主备模式，master机器处理所有写请求，Slave通过异步复制获取最新请求，提供读取服务）；
zookeeper引入Leader、Follower和Observer三种角色；leader提供读写服务，Follower和Observer提供读取服务，唯一区别是Observer不参与Leader选举过程，也
不参与写操作的“过半写成功”策略.
>* 会话————zookeeper默认对外服务端口2181，客户端启动时，会与zookeeper服务器建立一个TCP长连接，通过这个连接，客户端可以通过心跳检测与服务器保持有效会话，
也可以向服务器发送请求并接收参数，还可以接收来自服务器的watch事件通知.断开时在超时规定时间内重连会话仍有效.
>* 数据节点（Znode)————zookeeper中znode分为持久节点和临时节点.临时节点生命周期与客户端会话绑定,会话失效临时节点会被移除.znode上维护一个stat结构，
其中包括数据更改、ACL更改和时间戳的版本号，以允许缓存验证和协调更新.
>* 版本————对应znode的版本.
>* watcher————事件监听器，zookeeper允许用户在指定节点上注册一些watcher，并且在一些特定事件触发时服务器通知感兴趣的客户端.
>* ACL————Acess Control Lists.有5种权限：create、read、write、delete、admin.

1.4 简单的API.
>* create: 在树中的某个位置创建节点
>* delete: 删除节点
>* exists: 测试某个节点是否存在
>* get data
>* set data
>* get children
>* sync