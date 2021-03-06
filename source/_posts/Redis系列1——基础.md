---
title: Redis系列1——基础
date: 2020-02-25 21:04:00
categories: redis
tags: redis
---
redis简介.
>简单来说，redis就是一个基于内存的数据库；因为读写非常快，高性能且高并发，所以被广泛用于缓存方向；redis用多种数据类型来支持不同的业务场景；
>redis支持事务、持久化、lua脚本、LRU驱动事件、多种集群方案；

###1. redis单线程为什么这么快.
>1. 纯内存；
>2. 非阻塞IO；
>3. 单线程避免了线程切换和竞争消耗；

###2. redis特性.
>* 速度快；
>* 持久化；
>* 多种数据结构；
>* 支持多种编程语言；
>* 功能丰富；
>* 简单；
>* 主从复制；
>* 高可用，分布式；

###3. 应用场景.
>* 缓存系统；
>* 计数器；
>* 实时系统；
>* 排行榜；
>* 社交网络；
>* 消息队列系统；

###4. Redis数据结构
>Redis支持丰富的数据结构，常用的如string、list、set、zset、hash等；redis使用对象来表示K和V，Redis中的一个
>键值对，至少需要创建两个对象，一个键对象，一个值对象；

redis底层数据结构：
>SDS简单动态字符串：SDS与C字符串的比较：
>1. 获取字符串长度，SDS时间复杂度O(1)，因为内部存储了length；
>2. SDS不会发生内存溢出问题； 内部动态扩展；
>3. SDS减少内存分配次数； 会分配额外空闲空间；
>4. SDS是二进制安全的；

>链表，redis的链表特性：
>1. 无环双向链表；
>2. 获取表头指针、表尾指针、链表长度O(1)；
>3. 链表使用void*指针保存节点值，可以保存各种不同类型的值；

>哈希表，也叫字典；redis的哈希表中有两个哈希表，用于渐进式rehash；

>跳跃表；shiplist；redis的跳表实现由zskiplist和zskiplistNode组成，zskiplist保存跳表的信息（表头、表尾、长度），
>zskiplistNode表示跳表中的节点；

>整数集合，set的底层数据结构之一；

>压缩列表，list和hash的底层实现之一；是由一系列特殊编码的连续内存块组成的顺序性数据结构；
>

###4. 基本数据类型.
>* string字符串；可以是字符串、整形、浮点型； SDS字符串；
>* list；一个双向链表，链表的每个节点上都包含一个字符串； 快速链表=链表+压缩链表；
>* set；包含字符串的无序收集器；
>* zset；字符串成员与浮点数分值之间的有序映射，元素的排列顺序由分值的大小确定； 跳表，其实就是多层链表；
>>当数据量少时，由一个ziplist实现；当数据量多时，由一个dict+skiplist实现，dict用来查数据到分数的对应关系，
>而skiplist用来根据分数查询数据；
>* hash；包含键值对的无序散列表； redis是渐进式rehash的；

redis的rehash过程：
>在对哈希表进行扩展或者收缩操作时，rehash过程不是一次完成的，因为如果数据量过大时，一次性rehash会有庞大的
>计算量，这很可能导致服务器一段时间内停止服务；

>redis的所有数据类型都被封装成对象，hash结构的对象内部有两个哈希表，一个用于存放真实的K-V数据，另一个用于扩容；
>1. 字典内维持一个索引计数器，用于表示是否在进行rehash；
>2. rehash期间对字典进行增删改查时，除了执行指令外，还会将h[0]中数据rehash到h[1]中；
>3. 字典操作不断进行，最终rehash完时，计数器设为-1；
>4. 渐进式rehash过程中，字典会同时使用两个哈希表，删改查操作会同时在两个哈希表上进行；新增只会在h[1]上新增，
>查询时，优先查h[0]，如果查不到查询h[1]；


redis对象的一些细节.
>1. 服务器在执行某些命令时，会先检查给定的键的类型；
>2. redis对象系统带有引用计数实现的内存回收机制；‘
>3. redis会共享0-9999的字符串对象；
>4. 对象会记录自己最后一次被访问时间，这个时间可以用于计算对象的空转时间；

###5. redis线程模型.
>redis内部使用文件事件处理器File event handler，这个文件事件处理器是单线程的，所以redis被称为单线程的；它采用IO多路复用机制监听
>多个socket，根据socket上事件来选择对应的事件处理器进行处理；
>
>主要有两类事件：文件事件，就是对Socket操作的抽象；时间事件，就是对定时操作的抽象；
>
>这个文件事件处理器包含4个部分：
>* 多个socket；
>* IO多路复用程序；
>* 文件事件分派器；
>* 事件处理器；

###6. redis与Memcached的区别.
>1. redis支持更丰富的数据类型；Memcached只有string类型；
>2. redis支持持久化；Memcached不支持；
>3. memcached原生不支持集群，而redis支持集群；
>4. memcached是多线程、非阻塞IO复用的网络模型，Redis是单线程IO多路复用模型；

###7. redis过期时间和淘汰机制；
>redis可以设置expire time过期时间；redis使用定期删除+惰性删除维护key，注意定期删除是随机抽取的；
>如果定期删除漏掉了很多过期key，也没及时去查（没走惰性删除），大量key堆积导致内存耗尽，此时就需要内存淘汰机制；
>
>redis内存淘汰机制：
>* volatile-lru：从已设置过期时间的数据集（server.db[i].expires）中挑选最近最少使用的数据淘汰；
>* volatile-ttl：从已设置过期时间的数据集（server.db[i].expires）中挑选将要过期的数据淘汰；
>* volatile-random：从已设置过期时间的数据集（server.db[i].expires）中任意选择数据淘汰；
>* allkeys-lru：当内存不足以容纳新写入数据时，在键空间中，移除最近最少使用的key（这个是最常用的）；
>* allkeys-random：从数据集（server.db[i].dict）中任意选择数据淘汰；
>* no-eviction：禁止驱逐数据，也就是说当内存不足以容纳新写入数据时，新写入操作会报错。一般不用；
>* volatile-lfu：从已设置过期时间的数据集(server.db[i].expires)中挑选最不经常使用的数据淘汰；
>* allkeys-lfu：当内存不足以容纳新写入数据时，在键空间中，移除最不经常使用的key；
>
>使用redis时，为了提高缓存命中率，需要保证缓存数据都是热点数据，可以将内存最大使用量设置为热点数据占用的内存量，
>然后启用allkeys-lru策略；

###8. redis持久化机制.
>* 快照RDB.Redis可以通过创建快照来获得存储在内存里面的数据在某个时间点上的副本。创建快照的几种方式：
>>* BGSAVE命令； 创建出一个子进程来创建RDB文件，服务器进程可以继续服务；
>>* SAVE命令； 会阻塞Redis服务器进程，服务器不能接收任何请求，直到RDB文件创建完毕；
>>* save选项；
>>* SHUTDOWN命令；
>>* 一个redis服务器连接另一个redis服务器；
>* 只追加文件AOF.开启AOF持久化后每执行一条会更改Redis中的数据的命令，Redis就会将该命令写入硬盘中的AOF文件。

重写/压缩AOF.
>AOF虽然在某个角度可以将数据丢失降低到最小而且对性能影响也很小，但是极端的情况下，体积不断增大的AOF文件很可能会用完硬盘空间。另外，如果AOF体积过大，那么还原操作执行时间就可能会非常长。
>为了解决AOF体积过大的问题，用户可以向Redis发送 BGREWRITEAOF命令 ，这个命令会通过移除AOF文件中的冗余命令来重写（rewrite）AOF文件来减小AOF文件的体积。
>BGREWRITEAOF命令和BGSAVE创建快照原理十分相似，所以AOF文件重写也需要用到子进程，这样会导致性能问题和内存占用问题，和快照持久化一样。更糟糕的是，如果不加以控制的话，AOF文件的体积可能会比快照文件大好几倍。

RDB和AOF对过期键的策略.
>执行SAVE或BGSAVE创建RDB文件时，过期键不会保存到RDB文件中；载入RDB文件时，过期的键会被忽略；

>写入AOF文件时，过期键还没被定期/惰性删除时，不会影响写入AOF；载入AOF文件时，过期的键会被忽略；

>复制模式下，主服务器来控制从服务器统一删除过期键；

###9. Redis事务.
>Redis通过MULTI、EXEC、WATCH等命令实现事务功能；事务通过一次性将多个命令打包，然后一次性、顺序执行；redis事务总是具有ACI特性；
>
>redis同一个事务中如果有一条命令执行失败，其后的命令仍然会被执行，没有回滚。

###10. 缓存雪崩、缓存穿透、缓存击穿。
>* 缓存雪崩. 缓存同一时间大面积的失效，或者redis挂掉了，后面的请求都会落到数据库上，造成数据库短时间内承受大量请求而崩掉。
>解决方案.一般用设置随机过期时间；
>>* 事前：实现redis的高可用：主从+哨兵
>>* 事中：利用本地缓存和Hystrix限流；
>>* 事后：redis持久化，快速启动恢复缓存数据；
>* 缓存穿透. 大量无效key请求，直接请求数据库，导致数据库崩溃；解决方案：缓存无效key，校验，布隆过滤器；
>* 缓存击穿. 同一个key大量请求，某个时间点key过期，大量请求会直接到数据库；解决方案：设置热点key永不过期；

###11. Redis常见的性能问题和解决方案.
>1. Master写内存快照，save命令调度rdbSave函数，会阻塞主线程的工作，当快照比较大时对性能影响非常大，会间断性
>暂停服务，所以Master最好不要写内存快照；
>2. Master AOF持久化，如果不进行AOF重写，AOF文件会过大而影响Master重启速度；Master不要做任何持久化工作，由
>某个Slave开启AOF备份数据，策略是每秒一次；
>3. Redis主从复制的性能问题，为了主从复制的速度和连接的稳定性，Master和Slave最好在同一个局域网中；
>4. 尽量避免在压力很大的主库上增加从库；
>5. 主从复制不要用图状结构，用单向链表结构更稳定；例如Master < Slave1 < Slave2 ...这样结构方便解决单点故障
>问题，实现Slave对Master的替换。如果Master挂了，可以立刻启用Slave1，其他不变；

###12. Redis同步机制.
>主从同步。第一次同步时，主节点做一次bgsave，并同时将后续修改操作记录到内存buffer，待完成后将rdb文件全量同步
>到复制节点，复制节点接受完成后将rdb镜像加载到内存。加载完成后，再通知主节点将期间修改的操作记录同步到复制节点
>进行重放完成同步过程；

###13. Redis集群.
>* Redis Sentinel高可用，在master宕机时会自动将slave提升成master，继续提供服务；
>* Redis Cluster扩展性，在单个redis内存不足时，使用Cluster进行分片存储；

