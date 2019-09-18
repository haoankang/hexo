---
title: Zookeeper系列2——详解Paxos算法和ZAB协议
date: 2019-08-29 13:59:24
categories: zookeeper
tags: zookeeper
---

## Paxos算法.
paxos算法目的是构建一个分布式一致性状态机；

这里有几个概念要整理清楚：paxos算法中的三个角色proposal、acceptor和learner，每个进程可能不止一个角色；即
假如有A进程，那么A可能有proposal和acceptor两个角色，那么A的proposal会和A的acceptor相互通信；假设系统已经
可以产生一个全局时钟的概念来确保提案编号的顺序性；

算法过程：类似于二阶段提交，有两个阶段：
>perpare阶段：(a) Proposer选择一个提案编号N，然后向半数以上的Acceptor发送编号为N的Prepare请求。
 
>(b) 如果一个Acceptor收到一个编号为N的Prepare请求，如果小于它已经响应过的请求，则拒绝，不回应或
回复error。若N大于该Acceptor已经响应过的所有Prepare请求的编号（maxN），那么它就会将它已经接受过
（已经经过第二阶段accept的提案）的编号最大的提案（如果有的话，如果还没有的accept提案的话返回
{pok，null，null}）作为响应反馈给Proposer，同时该Acceptor承诺不再接受任何编号小于N的提案。
 
>accept阶段：(a) 如果一个Proposer收到半数以上Acceptor对其发出的编号为N的Prepare请求的响应，那么它
就会发送一个针对[N,V]提案的Accept请求给半数以上的Acceptor。注意：V就是收到的响应中编号最大的提案的
value（某个acceptor响应的它已经通过的{acceptN，acceptV}），如果响应中不包含任何提案，那么V就由
Proposer自己决定。

> (b) 如果Acceptor收到一个针对编号为N的提案的Accept请求，只要该Acceptor没有对编号大于N的Prepare
请求做出过响应，它就接受该提案。如果N小于Acceptor以及响应的prepare请求，则拒绝，不回应或回复error
（当proposer没有收到过半的回应，那么他会重新进入第一阶段，递增提案号，重新提出prepare请求）。

以上是paxos算法过程，这里自己整理不好语言，借鉴于网络；个人感觉是逻辑比较清晰的说明了；这是算法核心，通常
会有一些优化，例如选取主proposal保证算法活性，即完整算法过程只进行一次，之后就类似于master-slave模式了；

## ZAB协议.
ZAB协议目的是构建一个高可用的分布式数据主备系统；

ZAB协议有两种模式：崩溃恢复和消息广播，也就是两个过程，进一步分为三个阶段：发现、同步和广播；