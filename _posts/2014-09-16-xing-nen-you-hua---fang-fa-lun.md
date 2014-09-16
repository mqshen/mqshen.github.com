---
layout: post
title: "性能优化 方法论"
description: ""
category: ""
tags: []
---
{% include JB/setup %}

###术语
####IOPS
Input/Output Operations Per Second 即每秒钟读写操作的次数。
####Throughput 吞吐量
单位时间内成功传送数据的数量。在有些场合（如数据库）吞吐量能表示出操作数量。。。。
####Response time 相应时间
操作完成需要的时间。包括等待时间，处理时间，传送时间。
####Latency 延迟
等待处理时间。在有些情况下可以等同于response time
####Utilization 利用率
资源的使用率
####Saturation 饱和度
资源正在被排队使用情况
####Bottleneck 瓶颈
限制系统性能的资源
####Workload 工作量
系统所要处理的操作量
####Cache 缓存
可以存放有限数据的存储

###概念
####Latency 延迟
在一些环境延迟是性能的唯一目标。在另外的环境中延迟和吞吐量并称为最重要的两个指标。
####时间尺度

####权衡
我们经常要在 good/fast/cheap 中选择两个做取舍。
一个常用的取舍是在CPU和内存之间。内存可以缓存数据来减少CPU使用率。在一个CPU资源丰富的系统中可以可以压缩数据来减少内存的使用。
可调参数一般是用来做权衡的。如：
####File system record size
小的，接近系统I/O大小的块大小在随机读写和应用缓存中表现良好。
大的块大小可以提高流负载.
####Network buffer size
小缓存可以减少每个连接所使用的内存大小。
大缓存可以提高系统吞吐量


