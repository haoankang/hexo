---
title: xxl-job调度框架分析
date: 2019-06-13 20:11:15
categories:
tags: 调度框架
---

## xxl-job调度框架分析
###1. 简介
>xxl-job是一个一个开源的轻量级分布式调度平台，主要优点有开发迅速、学习简单、轻量级、易扩展、开箱即用、文档丰富、维护良好；
这里主要是分析源码，了解底层细节，也作为借鉴学习；
###2. 架构设计
>系统包含两部分：调度中心和执行器；<p>
将调度行为抽象形成"调度中心"公共平台xxl-job-admin，负责发起调度请求，不承担业务逻辑；执行器管理任务，将任务抽象成分散
的jobHandler，执行器负责接收调度请求并执行对应的JobHandler中业务逻辑；<p>
[官方文档地址](http://www.xuxueli.com/xxl-job/#/) 建议按照官方文档的指导快速入门，架构图如下
![架构图](https://raw.githubusercontent.com/xuxueli/xxl-job/master/doc/images/img_Qohm.png)
###3. 源码分析
#####1. 执行器
>执行器作为接入部分，先分析执行器中具体做了什么。<p>
执行器中一般要加载配置XxlJobExecutor属性，然后根据不同的接入方式，调用xxlJobExecutor的start()方法;
start()方法中的逻辑如下图：
{% plantuml %}
start
:将定义的JobHandler加载到ConcurrentHashMap<String, IJobHandler> jobHandlerRepository中;
:如果spring-boot方式接入，创建SpringGlueFactory;
:初始化日志路径logpath;
:初始化admin-client的调用类AdminBiz，将对应的XxlRpcReferenceBean放到adminBizList中;
:初始化日志清理线程，如果配置的清理时间小于3天，则不清理，否则每天定时清理;
:初始化回调线程TriggerCallbackThread;
:初始化执行器服务器;
stop
{% endplantuml %}
>其中逻辑比较复杂的是初始化执行器server，其步骤如下：
>* 根据一些规则得到当前执行器的地址——ip:port格式;
>* 初始化netty服务器（默认使用netty-http服务器），将执行处理器ExecutorBiz注入，安全校验;
>* netty服务器的请求处理pipeline中添加了NettyHttpServerHandler，这个类主要两个功能：1. 根据请求/services返回所有ExecutorBiz;2. 正常方法调用，采用java反射机制；
>* 启动ExecutorRegistryThread线程，自动注册服务(写入数据库);<p>

>销毁时，注销服务，关闭已启动的线程；
#####2. 调度中心
>调度中心是一个web项目，使用spring boot框架，前端采用freemarker框架，2.1.0之前的版本使用quartz的定时调度功能；<p>
调度中心的启动初始化在XxlJobDynamicSchedulerConfig中：配置和启动quartz的SchedulerFactory，调用XxlJobDynamicScheduler的start()方法；
初始化I18n，启动执行器注册监控线程（一个执行器可以有多个实例），启动失败日志重试和告警线程，初始化rpc服务；<p>
调度有两种方式：cron定时调度和手动执行调度，最终走同一个接口JobTriggerPoolHelper.trigger(..)，调度有几种方式：手动、cron、parent、失败重试和API，下面是流程图：
{% plantuml %}
start
:选择执行线程池（快线程池和慢线程池，主要区别在于队列大小等；如果一个任务连续超时10此，则放到慢线程池中执行）;
:判断如果传入的执行参数不为空，则覆盖之前的执行参数;
:设置失败重试次数;
:如果分片参数不为空，设置分片参数;
if(分片参数为空，且路由策略是分片路由)then(true)
:针对每个注册的执行器都执行;
else(no)
:根据路由策略找执行器执行;
endif
:执行方法processTrigger(..),先设置阻塞策略、路由策略和分片参数;
:数据库新增执行log记录;
:初始化TriggerParam作为rpc调用参数;
:根据路由策略选取执行器地址;
:调用执行器执行;
:收集调度信息并保存用于日志查看调度备注;
end
{% endplantuml %}
>核心部分在于调用rpc执行器执行，利用java动态代理机制获取代理类执行；代理类执行实际是通过服务器客户端
发送请求，默认使用netty-http客户端;
#####3. 执行器执行过程分析
>经过上面分析，我们知道：调度中心实际是调度执行器来执行，**阻塞策略、运行模式和任务超时等参数在执行器中生效**，下面
我们来分析执行器具体执行过程：
>* 根据选择的运行模式，获取JobHandler，如果是Glue模式，则动态加载类，如果是script，则构造相应的执行类；
>* 创建jobThread执行并注册缓存；
>* 将执行参数放到执行线程中；jobThread中的执行逻辑分析下：
>>1. 先执行jobHandler的init()方法，这里也是作者留的扩展点;
>>2. 循环标志toStop（线程一直存在，任务队列放置任务，手动停止），线程超时时间90s没收到请求参数，则关闭当前空闲执行线程；
>>3. 设置当前日志logFileName，判断任务是否设置超时时间，执行任务；
>>4. 任务执行完（成功或失败）或超时，任务结果回调，修改任务日志状态等;
>>5. 调用jobHandler的destory()方法。
#####4. 一些细节;
>1. 任务的超时和失败重试机制：任务执行状态最终会回调记录在log表中，调度中心启动时开启了一个监控线程，从表中
不断拉取失败任务，如果有重试次数，则再次调度，直到重试次数用完；且如果设置了告警邮箱，则失败重试时发送告警邮件；
>2. 子任务：任务执行完回调过程中，如果任务执行成功，且存在子任务，则执行子任务；
>3. 任务参数作为JobHandler执行参数可以传递参数变量；
>4. 日志系统：每次任务执行作为一个运行日志，JobHandler需要使用自定义的XxlJobLogger记录日志，分文件记录，
从而实现每个任务都有独立的执行日志；
#####5. 优点和总结
>1. 框架这种抽取基础公共类单独作为jar包，和spring-core类似，作为框架的常用方案；
>2. 完善的关闭清理机制；
>3. 反射+代理+服务器(netty)做RPC服务，细节值得借鉴；
>4. 功能模块分离思路可以学习；
>5. DB锁保证一致性和异步可以了解下；
>3. 待续..