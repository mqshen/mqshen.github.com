---
layout: post
title: "Zookeeper概念和基础"
description: "Zookeeper概念和基础"
category: "Zookeeper"
tags: [distribute, zookeeper]
---
{% include JB/setup %}
Zookeeper 分布式服务框架是 Apache Hadoop 的一个子项目，它主要是用来解决分布式应用中经常遇到的一些数据管理问题，如：统一命名服务、状态同步服务、集群管理、分布式应用配置项的管理等。    
它主要提供    
1.强一致性，排序，持久化保证    
2.实现经典同步原语的能力    
3.以一种更简单的方式来处理分布式系统中的并发问题  

使用单独的协调组件的有点：    
1.协调组件可以供很多应用使用    
2.使系统在协调方面的架构更简单    
3.分开运行的组件使得在生产中能更好的解决问题    

分布式系统中的进程一般使用两种方式来进行通讯：    
1.通过网络来直接交换信息    
2.从共享的存储中读写信息    
Zookeeper使用后者来实现协调和同步原语。

Znodes的两种不同模式    
1.持久化的--当它的创建不再是系统的一部分时我们还需要znode存储的信息我们就需要采用这种模式。（如：即使在master奔溃是我们也需要任务的分配信息）     
2.暂时的--只有在创建它的session有效时它才存在。（如：/workers下的znode当它不可用时消失）    
由于暂时的znode在session超时时就删除，所以不能包含子znode.(可能在将来的版本支持包含子znode)

Session time out    
假设会话超时时间为t，当客户端在t/3时间内没有收到服务端的消息，它就会相服务器发送心跳信息。当到达2t/3时间时开始寻找另外一台Zookeeper服务器，它还有t/3的时间去找到它。

每次Zookeeper的状态变化都是按顺序进行的。    


server.1={host}:{quorum通讯端口}:{leader选举端口}


