---
title: "Zookeeper系列4——Zookeeper的原理"
date: 2019-09-10 10:17:11
categories: zookeeper
tags: zookeeper
---

##1. 系统模型
1.1 数据模型
>zookeeper的数据节点ZNode按层次化结构组织，形成一棵树，节点的路径类似于linux文件系统，例如/node1/node2.

1.2 节点特性
>节点有两个维度，按照生命周期分为持久节点和临时节点，按照是否有序分为顺序节点和普通节点，组合产生了zookeeper
一共有四种节点类型：持久节点(PERSISTENT)、持久顺序节点(PERSISTENT_SEQUENTIAL)、临时节点(EPHEMERAL)、
临时顺序节点(EPHEMERAL_SEQUENTIAL),节点的属性如下图所示：
![节点属性](/images/zookeeper_1.png)

1.3 版本
>在zookeeper中，版本主要是用来做乐观锁的操作校验；下面是zookeeper更新前置校验代码：
```
private static int checkAndIncVersion(int currentVersion, int expectedVersion, String path)
        throws KeeperException.BadVersionException {
    if (expectedVersion != -1 && expectedVersion != currentVersion) {
        throw new KeeperException.BadVersionException(path);
    }
    return currentVersion + 1;
}
```

1.4 watcher机制
> 所有读取操作可以设置Watch，也就是getData(), getChildren(), and exists();zookeeper的watch定义如下：
watch事件是一次性触发器，发送给设置watch的客户端，当设置watch的数据发生更改时发生。重点是三个要点：一次性
触发、发送给客户端、watch是谁触发（data watches和child watches）；与此相关的有两个类：WatchedEvent和
WatcherEvent，后者实际上是前者的简化用于传输，主要包含三个信息：连接状态、触发事件类型、路径；

> watch的触发事件分为下面几个：
>* NodeCreated: Enabled with a call to exists.
>* NodeDeleted: Enabled with a call to exists, getData, and getChildren.
>* NodeDataChanged: Enabled with a call to exists and getData.
>* NodeChildrenChanged: Enabled with a call to getChildren.
>* ChildWatchRemoved: Watcher which was added with a call to getChildren.
>* DataWatchRemoved: Watcher which was added with a call to exists or getData.

> 工作机制.<br>
> zookeeper的watcher机制，总的来说分为三个阶段：客户端注册Watcher、服务端处理Watcher、客户端回调Watcher；
>>* 客户端注册watcher：客户端watcher经过包装构成了一个WatchRegistration对象，标记request，但最终打包packet发消息时
并没有携带此对象（包含两个字段：watcher和clientpath，可以理解成一个映射表），响应成功后，将WatchRegistration对象
放到对应的ZKWatchManager中进行管理；
>>* 服务端处理watcher——完成watcher注册：服务端接收到请求后，判断是否有watcher，如果有则将当前的ContextCnxn传递给getData
逻辑，因为继承了Watcher，可以看作是一个Watcher，数据节点的节点路径和ContextCnxn最终会被存储在WatchManager
的watchTable和watch2Paths中. 
>>* 服务端处理watcher——watcher触发：上一步可以看出，实际上相当于服务端也维护了一个映射表，这里注册的是getData()
方法，当调用setData(..)或delete(..)时会触发，服务端这类操作会在最后调用dataWatches.triggerWatch(path,event)
方法；无论是dataWatches或childWatches，触发执行逻辑是一样的：封装WatchedEvent，查询Watcher，调用process()
触发Watcher（组装发送一个通知给客户端）；以上步骤，所以真正的客户端触发回调和业务逻辑执行都在客户端；
>>* 客户端回调Watcher：客户端收到请求后判断这是一个watcher通知，再按照下面逻辑执行：反序列化，处理chrootpath，
还原WatchedEvent，回调Watcher. 以上回调过程，注意回调是放到队列中串行执行的；

1.5 ACL
> ACL(Access control list), Zookeeper的权限模式有三个部分：模式、授权对象和权限，通常使用
"scheme:id:permission"来标识一个有效的ACL信息.权限模式常用的有四种：IP、Digest、World、Super；
权限有五种：create、read、delete、write、admin.<br>
自定义权限控制器：实现接口org.apache.zookeeper.server.auth.AuthenticationProvider，再配置
authProvider.1=**.**.**（实现接口的类）

##2. 序列化与协议.
2.1 zookeeper的序列化协议是jute. 序列化和反序列化步骤如下：
>* 实体类需要实现Record接口的serialize和deserialize方法;
>* 构建一个序列化器BinaryOutputArchive;
>* 序列化；
>* 反序列化；

2.2 zookeeper有自己的通信协议.
>* 为了高效，zookeeper的协议尽可能简洁，包含三部分：length、请求头/响应头、请求体/响应体；请求头包含最
基本信息：xid和type，每个操作对应不同的请求体，格式相对固定；响应头包含每个响应最基本信息：xid、zxid和
err，不同响应类型对应不同响应体；

##3. 客户端.
3.1 客户端核心几个类：
>* Zookeeper实例：客户端入口；
>* ClientWatchManager：客户端watcher管理器；
>* HostProvider：客户端地址列表管理器；
>* ClientCnxn：客户端核心线程，内部两个线程，SendThread是I/O线程，负责客户端和服务端的所有网络通信，
EventThread是事件线程，负责对服务端事件进行处理；
![客户端整体结构](/images/zookeeper_2.png)

3.2 会话创建过程.
> 1. 初始化zookeeper对象；
> 2. 设置会话默认Watcher；
> 3. 构造服务器地址列表管理器HostProvider；
> 4. 创建并初始化客户端网络连接器ClientCnxn；
> 5. 初始化SendThread和EventThread；以上是初始化阶段；
> 6. 启动SendThread和EventThread；
> 7. 根据服务器地址列表获取一个地址；
> 8. 创建TCP连接,ClientCnxnSocket；
> 9. 构造ConnectRequest请求；
> 10. 发送请求；以上是会话创建阶段；
> 11. 接收服务端响应；
> 12. 处理Response；
> 13. 连接成功，生成事件：SyncConnected-None；
> 14. 查询watcher，处理事件；

3.3 服务器地址列表
> 创建客户端时，会传入地址列表，通过ConnectStringParser解析器解析：解析chrooPath和保存地址列表；
chrooPath就是为每个会话设定单独的根目录（命名空间），例如写入的地址列表是"host1:2181,host2:2181,host3:2181/hak"，
当前会话根目录就是/hak；zookeeper获取地址策略类似于Round Robin，先随机打乱组成一个环，以后就顺序
遍历这个环，也可以实现自己的路由策略；

3.4 ClientCnxn：网络I/O
> ClientCnxn是客户端核心类，内部定义了很多重要的类如：Packet、SendThread和EventThread及其对象用于管理；
其中Packet是每次请求的包装类，细节可见源码；还有比较重要的变量是outgoingQueue和pendingQueue，前者是
发送请求队列，后者是服务端的响应等待队列；ClientCnxnSocket定义了底层Socket通信接口；

##4. 会话
4.1 Zookeeper的连接和会话就是客户端通过实例化Zookeeper对象来实现客户端与服务端创建并保持TCP长连接的过程；
>会话状态有Connecting、Connected和Close.

4.2 会话创建
>会话Session包含四个属性：SessionId（根据机器标识和当前时间根据一定算法得出），TimeOut、TickTime、isClosing；

##5. 服务器启动
5.1 服务器的整体架构如下图：
![服务器整体架构](/images/zookeeper_3.png)

5.2 单机版服务器启动流程
![](/images/zookeeper_4.png)

5.3 集群版服务器启动流程
![](/images/zookeeper_5.png)

5.4 集群版Leader和Follower服务器启动期交互过程
![](/images/zookeeper_6.png)

##6. Leader选举
6.1 选举是按照投票来选举的，这里先明确投票的内容：
>* id: 被推举的leader的SID值；
>* zxid: 被推举的leader的事务id；
>* electionEpoch: 逻辑时钟，用来判断多个投票是否在同一选举周期内；自增序列；
>* peerEpoch: 被选举的leader的epoch；
>* state: 当前服务器状态:LOOKING\LEADING\FOLLOWING\OBSERVING；

6.2 leader选举流程：两台机器可以互联时，每台试图找到一个leader，于是开始leader选举，过程如下：
>* 每个server会发出一个投票，第一次投票通常都是投自己；
>* 接收来自各个服务器的投票；
>* 处理投票：zxid->id比较选出leader，再次发出投票；
>* 统计投票；
>* 改变服务器状态；

##7. 服务器角色
zookeeper服务器角色有：Leader、Follower、Observer；其中Observer可以没有，需要指定时在服务器列表中配置，
如server.xx=ip:port1:port2:observer；zookeeper的服务器采用责任链模式处理请求；集群间通信消息类型分为
四类：数据同步，服务器初始化，请求处理，会话管理；
>* leader的主要工作：事务请求的唯一调度和处理者，保证集群事务处理的顺序性；集群内部各服务器的调度者；
>* follower的工作：处理客户端非事务请求，转发事务请求给leader；参与事务请求的投票；参与leader选举投票；
>* observer的工作：observer工作原理和follower一致，唯一不同的是observer不参与任何形式的投票；observer
通常用于不影响集群事务处理能力的前提下提升集群的非事务处理能力；

##8. 请求处理
zookeeper服务器对会话请求处理，大体有以下阶段：请求接收、会话创建、预处理、事务处理、事务应用和会话响应；
> zookeeper的请求主要分为事务请求和非事务请求；需要注意的是：事务请求由leader处理，当follower或
observer接收到事务请求时，会在预处理器中转发给leader处理；事务请求例如setData()，主要流程：sync
-->proposal-->commit，也就是二阶段提交；非事务请求例如getData()等，要注意watch监听事件的处理；

##9. 数据与存储
在zookeeper中，数据存储分为两部分：内存数据存储和磁盘数据存储.

9.1 内存数据
>zookeeper的数据模型是一棵树，DataTree和DataNode，DataTree中有个核心变量:
```
 /**
 * This hashtable provides a fast lookup to the datanodes. The tree is the
 * source of truth and is where all the locking occurs
 */
private final ConcurrentHashMap<String, DataNode> nodes =
    new ConcurrentHashMap<String, DataNode>();
```
>这个map存储了zookeeper服务器上所有节点，可以说对于zookeeper数据的所有操作底层都是操作这个map，另外，
为了便于实时访问和及时清理，DataTree还单独将临时节点保存起来：
```
/**
 * This hashtable lists the paths of the ephemeral nodes of a session.
 */
private final Map<Long, HashSet<String>> ephemerals =
    new ConcurrentHashMap<Long, HashSet<String>>();
```
>ZKDatabase中引用了DataTree作为变量，是zookeeper的内存数据库，负责管理zookeeper的所有会话、DataTree
存储和事务日志；ZKDatabase会定时向磁盘dump快照数据，并且在启动时，通过磁盘上的事务日志和快照数据文件
恢复成一个完整的内存数据库；

9.2 事务日志
> 事务日志存储：存储路径配置参数dataLogDir目录，事务日志文件名后缀是该日志的第一条zxid；
> 事务日志格式：事务操作时间、客户端会话ID、CXID、ZXID、操作类型、节点路径、节点数据内容；
> 日志截断：当非leader机器上记录的事务ID比leader的大，就需要将非leader机器上记录的大于leader的
日志文件；

9.3 数据快照
> 存储路径dataDir目录，文件名类似于事务日志文件；

9.4 初始化
> 服务器启动之后，会根据自身配置文件zoo.cfg初始化，数据初始化阶段将存储在磁盘上的数据文件加载到zookeeper
服务器内存中；这一阶段会获取该服务器最后的事务ID，用于投票选举leader；

9.5 数据同步
> leader选举完成后，需要数据同步leader于learner；数据同步开始时，leader会先获取learner服务器状态，
主要是currentEpoch和lastZxid；然后开始数据同步初始化：首先从内存数据库中提取出事务请求对应的提议缓存
队列：proposals，同时完成对以下三个zxid的初始化：
>* peerLastZxid: 该learner服务器最后处理的zxid；
>* minCommitedLog: leader服务器提议缓存队列中的最小zxid；
>* maxCommitedLog: leader服务器提议缓存队列中的最大zxid；
>然后根据三个值的比较，选择合适的数据同步方式：直接差异化同步（DIFF同步）、先回滚再差异化同步（TRUNC+
DIFF同步）、仅回滚同步(TRUNC同步)和全量同步（SNAP同步）。