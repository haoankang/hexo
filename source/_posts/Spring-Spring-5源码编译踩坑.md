---
title: 'Spring: Spring-5源码编译踩坑'
date: 2019-04-26 14:57:44
categories:
tags: spring
---

##1. cglib和objenesis的编译错误解决
###问题分析：
    为了避免第三方class的冲突，Spring把最新的cglib和objenesis重新打包(repack)了，在spring-core模块中引用，
    可以看到gradle自定义的task任务cglibRepackJar和objenesisRepackJar，因此要先执行这些task任务才能编译成功；
    网络上有些是直接引用这些jar包，虽然也可以，但因为版本会更新相关jar包不好找，另一方面也没必要额外引入；而且即使这个
    问题使用引入jar包方式解决，后面编译还是会有些测试类找不到无法编译通过；
###解决办法：
    既然知道了问题原因，解决办法也就是先执行自定义task，两种方式；
    1. 可以使用command方式执行；
       进入源码的根目录，例如我的是：C:\WorkSpaces\source_code\spring-framework-5.1.6.RELEASE
       然后执行gradle cleanEclipse idea或者gradle cleanEclipse eclipse，根据自己的idea工具选择，
       然后执行gradle cglibRepackJar和gradle objenesisRepackJar，后面编译遇到类似问题，先看gradle
       文件，找到对应的自定义task，这样执行；
    2. 一般使用idea工具，都会自带可视化页面；例如idea可以像图1这样做：
![](/images/spring_1.png)
 
    直接点击执行即可;
##2. AspectJ编译问题解决
###问题分析：
    问题现象是：编译提示找不到AnnotationCacheAspect等类符号，实际可以找到，发现是这样描述的：
    <code>public aspect AnnotationCacheAspect extends AbstractCacheAspect {...}</code>
    像aspect不是java支持的关键字，它只是AspectJ才认识的关键字，需要ajc.exe来执行编译.
###解决办法：
    1. 下载aspect的jar包，目前版本1.9.3
    2. 建议使用java -jar xxx.jar的方式安装AspectJ，本质是解压，这种方式有可视化界面，且会自动配置classpath之类的（maybe?);
    3. 为spring-aspect工程添加Facets属性，如图2.
![](/images/spring_2.png)
    4. 更改编译器，如图3.
 ![](/images/spring_3.png)