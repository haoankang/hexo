---
title: "pring-boot与quartz集成的一些问题记录"
date: 2019-06-21 18:15:44
categories:
tags: quartz
---

#1. quartz问题：This scheduler instance is still active but was recovered by another instance in the cluster. This may cause inconsistent behavior.
>今天遇到一个问题，系统环境情况：有一个调度系统，系统内部有自己的数据表，集成了quartz，使用了quartz的持久化方案，
也就是JobStoreTX;<p>
问题现象描述：<br>
>>部署了两套环境，配置了两个数据库，假设为A环境-数据库AA，B环境-数据库BB，一套开发环境一套测试环境，是要分开
的；在A环境中新增任务，也就是AA中系统表和quartz表新增记录没问题，但是在调度开始后，发现执行调度的记录都新增在BB数据库中，
并且日志一直报：This scheduler instance is still active but was recovered by another instance in the cluster. This may cause inconsistent behavior.

>原因和解决办法：
>>最后查出，是因为B环境quartz的配置文件中配置了AA数据库，导致B环境一直在执行调度任务，并且执行任务中一些系统记录
会生成在BB数据库中，因为B环境一直在调度任务，导致A环境报上面提示错误；
#2. 在spring boot的配置文件中集成quartz配置参数
>spring boot2.0以上版本可以直接在application*.yml中配置quartz.properties中参数了，格式如下：
```
spring:
  application:
    name: xxx
  datasource:
    driver-class-name: com.mysql.jdbc.Driver
    url: jdbc:mysql://xxx:3306/xxx?useSSL=false&useUnicode=true&characterEncoding=UTF-8
    username: xxx
    password: xxx
  quartz:
    properties:
      org:
        quartz:
          scheduler:
            instanceName: smQuartzScheduler
            instanceId: AUTO
            skipUpdateCheck: true
          threadPool:
            class: org.quartz.simpl.SimpleThreadPool
            threadCount: 5
            threadPriority: 5
          jobStore:
            class: org.quartz.impl.jdbcjobstore.JobStoreTX
            driverDelegateClass: org.quartz.impl.jdbcjobstore.StdJDBCDelegate
            useProperties: true
            tablePrefix: ML_QRTZ_
            isClustered: true
            clusterCheckinInterval: 20000
            misfireThreshold: 60000
            dataSource: sm
          dataSource:
            sm:
              driver: ${spring.datasource.driver-class-name}
              URL: ${spring.datasource.url}
              user: ${spring.datasource.username}
              password: ${spring.datasource.password}
              maxConnections: 5
              validationQuery: select 1 from dual
```
#3. logback中配置环境变量
在logback中配置spring的一些变量时，需要使用<springProperty>标签;