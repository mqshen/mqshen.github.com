---
layout: post
title: "读懂GC日志"
description: "为了充分利用垃圾收集,你需要经常看看子系统在做什么。除了基本的verbose:gc标记, 还有很多可以控制输出信息的选项。"
category: ""
tags: []
---
{% include JB/setup %}
为了充分利用垃圾收集,你需要经常看看子系统在做什么。除了基本的verbose:gc标记, 还有很多可以控制输出信息的选项。  
不管怎 样,会读日志以及了解影响GC的基本选项非常重要,因为有时候你可能没法用GUI工具。最常用的GC日志选项如下所示：

| *选项* | *效果* |
|:------|:------|
| -XX:+PrintGCDetails | 关于GC更详细的细节 |
| -XX:+PrintGCDateStamps | GC操作的时间戳 |
| -XX:+PrintGCApplicationConcurrentTime | 在应用线程仍然运行的情况下用在GC上的时间 |  
  
  
这些选项组合在一起时,会产生下面这种日志:

    2013-10-28T14:01:39.884+0800: [GC [PSYoungGen: 297694K->43634K(305856K)] 
    297766K->50890K(1004928K), 0.2877270 secs] 
    [Times: user=0.34 sys=0.06, real=0.29 secs]

把它分解,看看每一部分是什么意思

    <time>:[GC [<collector name>: <occupancy at start> -> <occupancy at start> (<total size>)] 
    <full heap occypancy at start> -> <full heap occupancy at end>(<total heap size>), <pause time> secs]

Times: user= sys, real [参见](http://stackoverflow.com/questions/556405/what-do-real-user-and-sys-mean-in-the-output-of-time1)

第一块是GC的发生时间,从JVM启动开始算,到发生时的秒数。然后是用来收集年轻代的 收集器名称(PSYoungGen)。接着是年轻代收集前后占用的内存,以及年轻代的总大小。接着 是反映完全收集情况的相同部分。

除了GC日志选项,还有一个选项如果不经解释可能会引起误解。用选项-XX:+PrintGC- ApplicationStoppedTime产生的日志是这样的:
    
    Application time: 7.0147610 seconds

这些并不一定指GC用了多长时间,而是指在一个从安全点开始的操作中,线程停了多长时间。这包括GC操作,但也包括其他安全点操作(比如Java 6中的偏向锁操作),所以没有十足把 握说这是指GC时长。


