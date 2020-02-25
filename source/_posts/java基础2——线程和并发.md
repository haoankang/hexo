---
title: '''java基础2——线程和并发'''
date: 2020-02-25 15:23:49
categories: java
tags: java
---
###1. 使用和创建线程.
三种方式：实现Runnable接口、实现Callable接口、继承Thread.

###2. 基础线程机制.
Executor并发框架，有三种基础的Executor：FixedThreadPool、SingleThreadExecutor、CachedThreadPool.

###3. 锁.
synchronized和ReentrantLock比较：
>* 锁的实现：synchronized是jvm实现的，ReentrantLock是jdk实现的；
>* 性能：新版本synchronized优化了很多，例如锁自旋、锁消除等手段， 性能已经相差不多；
>* 等待可中断：ReentrantLock可中断，synchronized不可中断；
>* 公平锁：公平非公平是对获取锁而言，ReentrantLock支持公平锁和非公平锁，synchronized只有非公平锁；
>* 锁绑定多个条件：ReentrantLock可绑定多个Condition对象.

wait()和sleep()区别：
>* wait()会释放锁，而sleep()不会；
>* sleep()是静态方法，wait()不是；
>* wait()需要唤醒，或超时唤醒；

线程状态：
新建、可运行、阻塞、无限期等待、限期等待、死亡.

###4. J.U.C--AQS.
AQS：抽象队列同步器，是java.util.concurrent包的核心，是构建并发工具类的框架；用于实现依赖于FIFO等待队列的阻塞锁
和相关同步器；

>* CountDownLatch. 用来控制一个或多个线程等待多个线程；核心方法countDown()、await()；
>* CyclicBarrier. 用来控制多个线程互相等待，只有所有多个线程到达后，程序才继续执行；核心方法有await()、reset();
与CountDownLatch的区别是：可以通过reset()方法循环调用，所以叫循环屏障；
>* Semaphore. 信号量，控制对互斥资源的访问线程数；核心方法acquire()、release()；
>* Exchange. 多个线程间交换数据；

###5. Java内存模型.
分为主内存和工作内存.

内存间的交互操作：
read、load、use、assign、store、write、lock、unlock.


###6. 多线程开发实践.
>1. 给线程命名，方便debug；
>2. 缩小同步范围，减少锁争用；
>3. 多用同步工具，少用wait()和notify()；
>4. 使用本地变量和不可变类保证线程安全；
>5. 多用并发集合类；
>6. 使用线程池创建线程；
