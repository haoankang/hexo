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
jdk动态代理基于反射生成新的代理类，必须实现接口；

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
流程说明：
>* 客户端（浏览器）发送请求，直接请求到DispacterServlet;
>* DispacterServlet根据请求信息调用HandlerMapping，解析请求对应的Handler;
>* 解析到对应的handler后，开始由HandlerAdapter适配器处理;
>* HandlerAdapter回根据handler来调用真正的处理器开始处理请求，并处理相应业务逻辑;
>* 处理器处理完业务后，会返回一个ModelAndView对象;
>* ViewResolver会根据逻辑View查找实际的View;
>* DispacterServlet把返回的Model传给view;
>* view返回给客户端;

重要组件说明：
>* 前端控制器DispacterServlet;
>* 处理器映射器HandlerMapping;
>* 处理器适配器HandlerAdapter;
>* 处理器Handler(Controller);
>* 视图解析器ViewResolver;
>* 视图View(jsp,freemarker..);

**5.拦截器和过滤器**
过滤器处于客户端和web资源之间，可以针对request和response做过滤或提前设置参数，然后再传入servlet；拦截器是一种AOP；区别如下：
>拦截器是基于java反射机制的，而过滤器是基于函数回调；
>拦截器不依赖web容器，而过滤器依赖web容器；
>拦截器作为spring的组件，能通过IOC容器管理，获取spring中各种资源，而过滤器不能；
>过滤器是在servlet规范中定义的，由servlet容器支持；拦截器是在spring容器内的，由spring框架支持；

拦截顺序：过滤doFilter前-->拦截器preHandle-->action处理-->拦截器postHandle-->过滤doFilter后 

**6.SpringMVC中的异常处理**
>1. 自定义Exception;
>2. 利用ControllerAdvice和ExceptionHandler全局捕获处理;
>3. 定义一个返回对象，包括状态码、message、data等；

**7. Spring中的设计模式**
>* 工厂模式：利用BeanFactory和ApplicationContext创建bean对象;
>* 单例模式：例如bean的作用域为singleton时;
>* 代理模式：AOP的原理;
>* 模板方法：例如JdbcTemplete等;
>*  观察者 ：Sping事件驱动模型;
>*  适配器 ：AOP的增强或通知时使用了适配器模式，Jpa适配不同数据库；