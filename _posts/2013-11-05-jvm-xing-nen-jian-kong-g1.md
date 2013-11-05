---
layout: post
title: "JVM性能监控(G1)"
description: "G1垃圾回收器的英文全称是 Garbage-First Garbage Collector （又被称作G1 GC），这是一个新型的垃圾回收器，由JDK7中的Java HotSpot VM 引入。"
category: ""
tags: [JVM, G1, performance]
---
{% include JB/setup %}
###概述
G1垃圾回收器的英文全称是 Garbage-First Garbage Collector （又被称作G1 GC），这是一个新型的垃圾回收器，由JDK7中的Java HotSpot VM 引入。
目的是为了在达到高的吞吐量的同时尽量缩短的暂停时间。在Java 7 update 4 之后开始全面支持G1.
G1为满足以下特性设计的：

>   可以像CMS收集器一样和应用线程并发执行
>   不在需要冗长的暂停时间来进行垃圾回收
>   更好的预测GC暂停时间
>   不能牺牲很多的系统吞吐量
>   不需要增加很多的Java堆

和CMS相比的一些特性

>   G1是压缩收集器。G1可以压缩来完全避免细粒度的空间分配。而是在区域上分配。这大大的简化了收集器和最大限度的消除碎片问题。
>   G1提供更可预测的垃圾收集暂停，并能指定希望达到的暂停时间目标。

老的垃圾收集器的内存结构如下图所示

![jvm 内存]({{ BASE_PATH }}/assets/images/HeapStructure.png)

G1的内存结构如下图所示


![jvm 内存]({{ BASE_PATH }}/assets/images/G1HeapStructure.png)

堆被分成的大小相同的块。
在进行垃圾回收时G1的操作与CMS相同。它会先收集快要为空的区域，集中压缩收集那些被可回收对象充满的区域。

###G1占用空间
Remembered Sets或RSet 它跟踪区域内的对象引用。堆中的每个区域都有一个RSet。大小不超过5%
Collection Sets或CSet 它跟踪在GC中需要收集的区域。CSet大小不超过1%

###推荐使用G1的情况
全收集消耗时间太长或太平凡
对象分配，升级比率变化明显
不希望太长的垃圾回收，压缩时间

### G1参数

| *参数* | *作用* |
|:-------|:-------|
| -XX:+UseG1GC | 启用G1 |
| -XX:MaxGCPauseMillis=n | 最大的GC暂停时间 |
| -XX:InitiatingHeapOccupancyPercent=n | 堆占用多少百分比之后开始GC周期,默认为45 |
| -XX:NewRatio=n | 新生代与老年代的比例,默认为2 |
| -XX:SurvivorRatio=n | eden与survivor的比列，默认为8 |
| -XX:MaxTenuringThreshold=n | 对象升迁的阈值，默认为15 |
| -XX:ParallelGCThreads=n | 垃圾回收并发阶段的线程数 |
| -XX:ConcGCThreads=n | 并发垃圾回收器的线程数 |
| -XX:G1ReservePercent=n | 设置需要保留的堆的大小，用来减少对象升迁失败的可能性 |
| -XX:G1HeapRegionSize=n | 设置G1区域块的大小， 1Mb - 32Mb |


#### 参考资料
>   [Getting Started with the G1 Garbage Collector](http://www.oracle.com/webfolder/technetwork/tutorials/obe/java/G1GettingStarted/index.html)
