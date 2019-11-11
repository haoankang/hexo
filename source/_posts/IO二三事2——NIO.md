---
title: IO二三事2——NIO
date: 2019-11-11 20:29:25
categories:
tags: I/O
---
## 一些基础
1. 什么是NIO.
   >NIO(Non-blocking I/O,也被称为New I/O),是一种同步非阻塞I/O模型(select/poll)，也是I/O多路复用的基础(epoll)，
   常用于解决高并发与大量连接的I/O问题；
2. NIO的核心.
    >NIO的核心是Buffer、Channel和selector；
    >* 缓冲区Buffer是一个数组对象，NIO库中，所有数据都是用缓冲区处理的；
    >* 通道Channel是双向的，网络数据通过Channel读取和写入，分为两大类：用于网络读写的SelectableChannel和用于文件操作的FileChannel；
    >* 多路复用器Selector，提供选择已就绪任务的能力；