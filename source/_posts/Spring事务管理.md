---
title: Spring事务管理
date: 2020-02-22 13:47:47
categories: Spring
tags: Spring
---
##1. 概述
事务是逻辑上的一组操作，要么都执行，要么都不执行；要满足ACID特性；

##2. Spring事务管理接口
>* PlatformTransactionManager：事务管理器
>* TransactionDefinition：事务定义信息（事务隔离级别、传播机制、回滚机制、超时、只读规则）
>* TransactionStatus：事务运行状态

Spring并不直接管理事务，而是提供多种事务管理器，将事务管理的职责委托给Hibernate和JPA等持久化
框架实现；

>并发事务带来的问题：
>* 脏写
>* 脏读
>* 不可重复读
>* 幻读

>事务隔离级别：
>* 读未提交TransactionDefinition.ISOLATION_READ_UNCOMMITTED
>* 读已提交TransactionDefinition.ISOLATION_READ_COMMITTED
>* 可重复读TransactionDefinition.ISOLATION_REPEATABLE_READ
>* 序列化TransactionDefinition.ISOLATION_SERIALIZABLE
>* 具体实现方式可以参考之前mysql事务部分

>事务传播机制：当一个事务方法被另一个事务方法调用时，必须指定事务如何传播
>* TransactionDefinition.PROPAGATION_REQUIRED： 如果当前存在事务，则加入该事务；如果当前没有事务，则创建一个新的事务。
>* TransactionDefinition.PROPAGATION_SUPPORTS： 如果当前存在事务，则加入该事务；如果当前没有事务，则以非事务的方式继续运行。
>* TransactionDefinition.PROPAGATION_MANDATORY： 如果当前存在事务，则加入该事务；如果当前没有事务，则抛出异常。（mandatory：强制性）
>* TransactionDefinition.PROPAGATION_REQUIRES_NEW： 创建一个新的事务，如果当前存在事务，则把当前事务挂起。
>* TransactionDefinition.PROPAGATION_NOT_SUPPORTED： 以非事务方式运行，如果当前存在事务，则把当前事务挂起。
>* TransactionDefinition.PROPAGATION_NEVER： 以非事务方式运行，如果当前存在事务，则抛出异常。

##3. Spring支持两种方式的事务管理
>* 编程式事务  通过Transaction Template手动管理事务，实际应用中很少使用
>* 声明式事务  推荐使用（代码侵入性最小），实际是通过AOP实现

