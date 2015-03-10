---
layout: post
title: "Mesos笔记"
description: ""
category: ""
tags: []
---
{% include JB/setup %}
这个是对[Mesos: A Platform for Fine-Grained Resource Sharing in the Data Center](https://amplab.cs.berkeley.edu/wp-content/uploads/2011/06/Mesos-A-Platform-for-Fine-Grained-Resource-Sharing-in-the-Data-Center.pdf)论文的一些笔记。

##前言    
Mesos的目标就是以一种合适的粒度在不同的Framework之间高效的共享资源的同时又能简化自身的调度逻辑，使其尽可能的保证兼容性和可扩展性，以保证在大规模集群使用环境下的健壮性和对各种可能的运算框架的普遍适用性。

##简介    
Mesos采用一种叫resource offers的两级的分布式资源调度机制。   
第一级，由mesos将资源分配给框架；    
第二级，框架自己的调度器将资源分配给自己内部的任务。    


Mesos只负责将可用资源提供给各个调度框架。具体的调度策略有Framework的调度逻辑来实现。    

Framework的调度器通过调用Mesos的调度器驱动器（MesosSchedulerDriver）中的接口与Mesos-master一系列交互（如：注册，资源分配等）    

每个Slave节点将本节点上的可用资源报告给Master节点，Master节点再将可用资源按一定的逻辑（如公平调度，优先级调度等）提供给各个Framework的调度进程。Framework再决定将这些资源部分或全部分配给特定的任务，同时把所分配的任务-资源关系列表返回给Master节点，Master节点进而将这些任务提交给对应的Slave节点执行。

Framework可以通过设置filters来更快的确定Mesos提供的资源是否满足Framework的需求。    

下图表示了一个简单的Mesos资源调度步骤：    
![调度](http://mesos.apache.org/assets/img/documentation/architecture-example.jpg)


##架构
![架构](http://mesos.apache.org/assets/img/documentation/architecture3.jpg)

Mesos为了简化设计，也是采用了master/slave结构，为了解决master单点故障，将master做得尽可能地轻量级，其上面所有的元数据可以通过各个slave重新注册而进行重构，故很容易通过zookeeper解决该单点故障问题。


Mesos由四个组件组成，分别是  *Mesos-master*，*mesos-slave*，*framework*和*executor*。    

*Maste* 是整个系统的核心，负责管理接入mesos的各个framework（由frameworks_manager管理）和slave（由slaves_manager管理），并将slave上的资源按照某种策略分配给framework（由独立插拔模块Allocator管理）。    

*Slave* 负责接收并执行来自mesos-master的命令、管理节点上的mesos-task，并为各个task分配资源。mesos-slave将自己的资源量发送给mesos-master，由mesos-master中的Allocator模块决定将资源分配给哪个framework，当前考虑的资源有CPU和内存两种，也就是说，mesos-slave会将CPU个数和内存量发送给mesos-master，而用户提交作业时，需要指定每个任务需要的CPU个数和内存量，这样，当任务运行时，mesos-slave会将任务放到包含固定资源的linux container中运行，以达到资源隔离的效果。    

*Framework* 是指外部的计算框架，如Hadoop，Mesos等，这些计算框架可通过注册的方式接入mesos，以便mesos进行统一管理和资源分配。    

*Executor* 主要用于启动框架内部的task。 由于不同的Framework启动task的接口或者方式不同，所以Mesos要求Framework通过调用Mesos的执行器驱动器（MesosExecutorDiver）中的接口来告诉mesos启动task的方法。
