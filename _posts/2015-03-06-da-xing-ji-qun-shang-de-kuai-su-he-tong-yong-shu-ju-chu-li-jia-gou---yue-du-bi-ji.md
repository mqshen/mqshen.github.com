---
layout: post
title: "大型集群上的快速和通用数据处理架构 阅读笔记"
description: "An Architecture for Fast and General Data Processing on Large Clusters"
category: "Spark"
tags: [Spark, Cluster]
---
{% include JB/setup %}
这个是对[Matei Zaharia](https://twitter.com/matei_zaharia)的论文[An Architecture for Fast and General Data Processing on Large Clusters](http://www.eecs.berkeley.edu/Pubs/TechRpts/2014/EECS-2014-12.pdf)的一些笔记

##弹性分布式数据集（Resillient Distributed Dataset）

###优点    
1.可以使用lineage来回复数据，这就不需要检查点开销。此外，当出现失败时，只有丢失的那部分RDD分区需要重新计算，而且计算可以在多个节点上并发完成，不必回滚整个程序。    
2.RDD的不可变性让它可以像MapReduce一样使用备份任务来替代运行缓慢的任务来减少缓慢节点。    
3.任务的执行可以根据数据的位置来进行优化，从而提高性能。    
4.只要所进行的操作是只基于扫描的，当内存不足时，RDD的性能下降也是平稳的。    

###接口    
所有RDD包含5种信息：    
1.一组分区--他们是数据集的最小分片    
2.依赖关系--指向其父RDD    

#####窄依赖    
每个父RDD的分区都至多被一个子RDD的分区使用

#####宽依赖    
多个子RDD的分区依赖一个父RDD的分

3.一个函数--基于其父RDD的运算    
4.划分策略--对于计算出来的数据结果如何分发(optional)    
5.数据位置--对于数据分区的偏好(optional)    

##调度器
当用户对一个RDD执行Action(如count,save)操作时，调度器会根据该RDD的lineage来构建一个由若干stage组成的一个有向无环图(DAG)以执行程序。
每个stage都包含尽可能多的连续的窄依赖型转换。各个阶段之剑的分界则是宽依赖所需的shuffle操作，或者是DAG中一个经由该分区能更快到达父RDD的已计算分区。    
当stage被提交之后，调度器会划分多个任务来计算各个阶段所缺失的分区.    
调度器根据*数据存储位置*向各机器*延时调度*任务。
