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
>>* 务端处理watcher——watcher触发：上一步可以看出，实际上相当于服务端也维护了一个映射表，这里注册的是getData()
方法，当调用setData(..)或delete(..)时会触发，服务端这类操作会在最后调用dataWatches.triggerWatch(path,event)
方法；无论是dataWatches或childWatches，触发执行逻辑是一样的：封装WatchedEvent，查询Watcher，调用process()
触发Watcher（组装发送一个通知给客户端）；以上步骤，所以真正的客户端触发回调和业务逻辑执行都在客户端；
>>* 客户端回调Watcher：客户端收到请求后判断这是一个watcher通知，再按照下面逻辑执行：反序列化，处理chrootpath，
还原WatchedEvent，回调Watcher. 以上回调过程，注意回调是放到队列中串行执行的；