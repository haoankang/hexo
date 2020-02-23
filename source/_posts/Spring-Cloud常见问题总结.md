---
title: Spring Cloud常见问题总结
date: 2020-02-23 10:48:48
categories: spring cloud
tags: spring cloud
---

##1. 什么是微服务.
>微服务是一种架构模式，是一种特殊的分布式，提倡将单一应用程序划分成一组小的服务，每个服务运行在自己的进程中，服务之间互相协作，互相配合，
>为用户提供服务；服务之间采用轻量级的通信机制互相沟通；

##2. 微服务优缺点.
优点：
>单一职责，代码容易理解，开发效率提高，松耦合，可扩展，高可靠；

缺点：
>架构复杂，运维复杂，通信成本；

##3. 什么是Spring Cloud.
Spring Cloud是基于Spring Boot的一套微服务解决方案，提供了包括服务注册和发现、配置中心、服务网关、负载均衡和路由、熔断器和链路监控等
组件；

##4. Spring Cloud和Dubbo.
>* Spring Cloud和Dubbo都是微服务架构；
>* Spring Cloud包含微服务所需的一系列组件，而Dubbo只是服务治理；
>* 服务调用方式：Dubbo使用的是RPC远程调用，而Spring Cloud使用的是Rest API;
>* 注册中心：Dubbo使用zookeeper作为注册中心，强调CP；Spring Cloud默认使用Eureka作为注册中心，强调AP；

##5. 微服务之间如何通信.
>* 同步：RPC或Rest等；
>* 异步：消息队列；

##6. Ribbon和Feign.
Ribbon和Feign都是客户端的负载均衡工具，Feign的底层是通过Ribbon实现的，它是对Ribbon的进一步封装；Ribbon使用HttpClient和RestTemplate
模拟http请求，而feign采用注解+接口方式，不需要自己构建http请求；

##7. Eureka的自我保护模式.
 默认情况下，如果EurekaServer在一定时间内没有接收到大部分微服务实例的心跳，EurekaServer将会进入自我保护模式，在该模式下EurekaServer
 就会保护服务注册表中的信息，不再删除服务注册表中的数据（也就是不会注销任何微服务）。当网络故障恢复后，该Eureka Server节点会自动退出
 自我保护模式。
 
##8. Eureka和ZooKeeper都可以提供服务注册与发现的功能,请说说两个的区别.
>* eureka保证的是AP，zookeeper保证的是CP；
>* zookeeper节点有leader和follow等角色，而eureka各个节点平等；
>* zookeeper采用过半存活原则，eureka采用自我保护机制解决分区问题；
>* eureka本质上是一个工程，而zookeeper是一个进程；
##9. 什么是服务熔断，什么是服务降级.
微服务架构中，服务之间相互调用形成调用链；当下游服务因为某些原因变得不可用或反应过慢，上游服务为了保证整体服务的可用性，不再继续调用
目标服务，快速返回释放资源，这种行为被称为熔断机制；熔断和降级一般配合使用，服务熔断是降级的一种方式；

##10. 服务雪崩.
微服务之间存在服务之间相互调用，当一个下游服务失败，导致整个链路的服务都失败，被称为服务雪崩；

##11. eureka集群怎么构建.
互相注册，主要配置修改如下，fetch-registry和registry-with-eureka要设置为true，并配置service-url；
```
eureka:
  instance:
    hostname: eureka1
  client:
    fetch-registry: true
    register-with-eureka: true
    service-url:
      defaultZone: http://localhost:8022/eureka/
```

##12. eureka中角色
eureka分为服务端和客户端，按照数据流向分为注册中心、服务提供者和服务消费者；主要由fetch-registry和registry-with-eureka参设设置；
>* 服务提供者启动后，向注册中心注册；运行过程中，定时向注册中心发送心跳；停止服务时，向注册中心发送cancel请求；
>* 服务消费者启动后，从注册中心拉取服务；运行过程中，定时更新注册表；
>* 注册中心启动后，从其他节点拉取注册信息；运行过程中，定时清理没有按时发送心跳的服务节点；运行过程中，收到registry、renew、cancel请求，
>都会同步到其他注册中心节点；

eureka数据存储结构
>分为两层：数据存储层和缓存层；eureka client在拉取服务时，会先从缓存层获取；
>* 数据存储层，是一个双层的ConcurrentHashMap结构；第一层key是spring.application.name，第二层是instanceId;
>* 缓存层，分为一级缓存和二级缓存；二级缓存定时更新一级缓存；

服务注册机制：
>1. 保存服务信息，将服务信息保存到registry中；
>2. 更新队列，将此事件添加到更新队列中，供 Eureka Client 增量同步服务信息使用。
>3. 清空二级缓存，即 readWriteCacheMap，用于保证数据的一致性。
>4. 更新阈值，供剔除服务使用。
>5. 同步服务信息，将此事件同步至其他的 Eureka Server 节点。

服务续约机制：
>1. 更新服务对象的最近续约时间，即 Lease 对象的 lastUpdateTimestamp;
>2. 同步服务信息，将此事件同步至其他的 Eureka Server 节点。

服务注销机制：
>1. 删除服务信息，将服务信息从 registry 中删除；
>2. 更新队列，将此事件添加到更新队列中，供 Eureka Client 增量同步服务信息使用。
>3. 清空二级缓存，即 readWriteCacheMap，用于保证数据的一致性。
>4. 更新阈值，供剔除服务使用。
>5. 同步服务信息，将此事件同步至其他的 Eureka Server 节点。

服务剔除条件：
>1. 关闭了自我保护
>2. 如果开启了自我保护，需要进一步判断是 Eureka Server 出了问题，还是 Eureka Client 出了问题，如果是 Eureka Client 出了问题则进行剔除。
这里判断使用阈值；

##13. Ribbon负载均衡.
Ribbon原理：在程序启动时，如果启用了Ribbon，Ribbon会从注册中心拉取服务列表，然后根据相应的策略选择节点；

