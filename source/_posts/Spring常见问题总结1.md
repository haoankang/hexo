---
title: '''Spring常见问题总结1'''
date: 2020-01-20 13:56:45
categories: spring
tags: spring
---

**1.Spring IOC&AOP**
IOC即依赖注入，是一种设计思想，将类的创建和依赖交给框架通过配置来管理，资源不由使用资源的双方管理而由第三方管理；可以降低耦合度，资源集中管理使得资源可配置和易管理；
IOC容器实际上就是个Map，Map中存放了各种对象；Spring IOC的初始化（实现）过程：
>>xml配置--（读取）-->Resource--（解析）-->BeanDefinition--（注册）-->BeanFactory

AOP即面向切面编程，可以为业务模块所共用的逻辑（如事务、日志、权限）封装起来，便于减少系统重复代码、降低耦合，有利于扩展和维护；Spring AOP是基于动态代理的，如果代理
对象实现了接口，使用jdk动态代理，如果没有实现接口则使用Cglib实现代理；

**2.Spring AOP和AspectJ AOP区别**
Spring AOP属于运行时增强，AspectJ是编译时增强。Spring AOP基于代理，而AspectJ基于字节码操作；Spring AOP集成了AspectJ，常用的aop注解是AspectJ的；

**3.Spring的生命周期**
{% plantuml %}
start
:实例化BeanFactoryPostProcessor实现类;
:执行BeanFactoryPostProcessor的postProcessBeanFactory();
:实例化BeanPostProcessor实现类;
:实例化InstantiationAwareBeanPostProcessorAdapter实现类;
:执行InstantiationAwareBeanPostProcessor的postProcessBeforeInstantiation();
:执行Bean构造器;
:执行InstantiationAwareBeanPostProcessor的postProcessPropertyValues();
:为Bean注入属性;
:调用BeanNameAware的setBeanName();
:调用BeanFactoryAware的setBeanFactory();
:执行BeanPostProcessor的postProcessBeforeInitialization();
:调用InitializingBean的afterPropertiesSet();
:调用<bean>的init-method指定的方法;
:调用BeanPostProcessor的postProcessAfterInitialization();
:执行InstantiationAwareBeanPostProcessor的postProcessAfterInitialization();
:容器初始化成功，执行正常调用后，下面销毁容器;
:调用DiposibleBean的destory方法;
:调用<bean>的destory-method指定的方法;
end
{% endplantuml %}

**4.Spring MVC**
SpringMVC是一个Web开发框架，把后端项目分层为Model、View、Controller，工作原理如下：
![](/images/spring_4.jpg)

