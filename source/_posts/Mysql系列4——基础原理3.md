---
title: '''Mysql系列4——基础原理3'''
date: 2020-01-02 20:03:19
categories: mysql
tags: mysql
---

**1.事务简介**
事务是作为单个逻辑工作单元执行的一系列操作；Spring事务本质上使用数据库锁，也就是数据库事务，Spring事务只有在方法执行过程中出现
异常才会回滚，并且只回滚数据库相关的操作；

事务特性ACID：原子性（Atomicity）、一致性（Consistency）、隔离性（Isolation）、持久性（Durability）；

mysql只有InnoDB和NDB存储引擎支持事务；开启事务：'begin;'或者'start transaction;'，显示提交事务就是commit；回滚事务rollback；
自动提交事务autocommit;隐式提交事务：当执行一些特殊语句而导致事务提交，例如ddl、隐式使用或修改mysql数据库中的表等等；保存点
的概念savepoint [savepointname]；

**2.事务并发执行遇到的问题**
>* 脏写：一个事务修改了另一个未提交事务修改过的数据；
>* 脏读：一个事务读取了另一个未提交事务修改过的数据；
>* 不可重复读：一个事务两次读取同一行数据，得到不同结果；
>* 幻读：一个事务中两次读取一个表数据，得到不同结果，或多行或少行；

**3.事务隔离级别**（下面只是一种实现方式）
>* Read Uncommitted: 事务对当前读取的数据不加锁，事务在更新某数据时，必须先对其加行级共享锁直到事务结束；
>* Read Committed: 事务对当前被读取的数据加行级共享锁，一旦读完立即释放行级共享锁；事务在更新某数据时先加行级排他锁直到事务结束；
>* Repeatable read: 事务在读取数据时先加行级共享锁直到事务结束；事务在更新时先加行级排他锁直到事务结束；
>* Serializable: 事务在读取数据时先加表级共享锁直到事务结束；事务在更新时先加表级排他锁直到事务结束；

隔离级别|脏读|不可重复读|幻读
---|---|---|---
读未提交|可能|可能|可能
读已提交|不可能|可能|可能
可重复读|不可能|不可能|可能
序列化|不可能|不可能|不可能

**4.MVCC**
multi-version concurrency control，多版本并发控制；因为加锁太消耗性能，MVCC是同一份数据临时保留多版本的一种方式实现并发控制；对于Mysql
的InnoDB来说，只有事务隔离级别是Read Committed和Repeatable Read时会使用MVCC保证不同事务的读写操作并发执行，不同点在于生成ReadView的时机
不同，Read Committed在每次执行普通select前生成一个，Repeatable read是在第一次进行普通select前生成一个ReadView，之后都是用这个；

ReadView中比较重要的内容：
>* m_ids:表示在生成ReadView时当前系统中活跃的读写事务的事务id列表；
>* min_trx_id:表示生成ReadView时，当前系统中活跃的读写事务中最小的事务id；
>* max_trx_id:同上，最大的事务id；
>* creator_trx_id:表示生成该ReadView的事务id；

访问某条记录，按下面步骤判断：
>* 如果被访问版本的trx_id属性值与ReadView中的creator_trx_id值相同，意味着当前事务在访问它自己修改过的记录，所以该版本可以被当前事务访问。
>* 如果被访问版本的trx_id属性值小于ReadView中的min_trx_id值，表明生成该版本的事务在当前事务生成ReadView前已经提交，所以该版本可以被当前事务访问。
>* 如果被访问版本的trx_id属性值大于或等于ReadView中的max_trx_id值，表明生成该版本的事务在当前事务生成ReadView后才开启，所以该版本不可以被当前事务访问。
>* 如果被访问版本的trx_id属性值在ReadView的min_trx_id和max_trx_id之间，那就需要判断一下trx_id属性值是不是在m_ids列表中，如果在，说明创建ReadView时生成该版本的事务还是活跃的，该版本不可以被访问；如果不在，说明创建ReadView时生成该版本的事务已经被提交，该版本可以被访问。

**5.锁**