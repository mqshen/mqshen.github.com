---
layout: post
title: "JVM性能监控(垃圾回收)"
description: "现代JVM实现（Java HosSpot VM)提供了以文本、log文件、监控界面等形式让我们直观的了解垃圾回收的统计信息。这对应用的吞吐量和延迟有着积极作用。"
category: ""
tags: [Java performance, Java, JVM]
---
{% include JB/setup %}

### 应该关注的垃圾回收数据

>   使用那种垃圾回收机制  
>   Java堆的大小  
>   新生代和老年代的大小  
>   永久代的大小  
>   次收集（minor GC)用时多少  
>   次收集（minor GC)的频率  
>   次收集（minor GC)能回收多少空间  
>   全收集（full GC)用时多少  
>   全收集（full GC)的频率  
>   一个并发垃圾回收周期能回收多少空间  
>   垃圾回收前后的Java堆的使用情况  
>   垃圾回收前后的新生代和老年代的使用情况  
>   垃圾回收前后的永久代的使用情况  
>   全收集（full GC)是由老年代还是永久代所出发的  
>   应用是否显示的调用了System.gc()  

### 垃圾回收报告
有两种不同的垃圾收集
次收集（minor GC)：回收新生代的空间
全收集（full  GC)：回收老年代和永久代的空间
这里有一些例外 HotSpot VM 的全收集默认是回收新生代，老年代，永久代的空间。  
关闭全收集是的新生代收集:

    -XX:-ScavengeBeforeFullGC

##### -XX:+PrintGCDetails
虽然说-verbose:gc可能是最常用的gc报告命令,但-XX:+PrintGCDetails能收集更多的垃圾收集信息。

    [GC [PSYoungGen: 297694K->43634K(305856K)] 
    297766K->50890K(1004928K), 0.2877270 secs] 
    [Times: user=0.34 sys=0.06, real=0.29 secs]

把它分解,看看每一部分是什么意思

    [GC [<collector name>: <occupancy at start> -> <occupancy at end> (<total size>)] 
    <full heap occypancy at start> -> <full heap occupancy at end>(<total heap size>), <pause time> secs]
    [Time: 经过的cpu时间] 

PSYoungGen表示正在使用的是为吞吐量优化的多线程的新生代垃圾收集。使用 –XX:+UseParallelGC 启用或使用 –XX:+UseParallelOldGC 自动打开。
其他可能的收集器有ParNew(CMS中使用的多线程新生代收集器）,DefNew(单线程的新生代收集器,由serial收集器使用)


    [Full GC
        [PSYoungGen: 11456K->0K(110400K)]
        [PSOldGen: 651536K->58466K(655360K)]
        662992K->58466K(765760K)
        [PSPermGen: 10191K->10191K(22528K)],
        1.1178951 secs]
        [Times: user=1.01 sys=0.00, real=1.12 secs]

把它分解,看看每一部分是什么意思

    [Full Gc
        [<collector name>: <occupancy at start> -> <occupancy at end>(<total size>)]
        [<collector name>: <occupancy at start> -> <occupancu at end>(<total size>)]
        <heap occupancy at start> -> <heap occupancy at end>(<total heap size>)
        [<collector name>: <occupancy at start> -> <occupancu at end>(<total size>)]
        elapsed time for the garbage collection]
        [Time: 经过的cpu时间] 

当使用concurrent garbage collector(CMS)这些输出会不一样
启用CMS

    -XX:+UseConcMarkSweepGC

    [GC
        [ParNew: 2112K->64K(2112K), 0.0837052 secs]
        16103K->15476K(773376K), 0.0838519 secs]
        [Times: user=0.02 sys=0.00, real=0.08 secs]


CMS整个垃圾回收周期日志

    [GC
        [1 CMS-initial-mark: 13991K(773376K)] 14103K(773376K), 0.0023781 secs]
        [Times: user=0.00 sys=0.00, real=0.00 secs]
    [CMS-concurrent-mark-start]
    [GC
        [ParNew: 2077K->63K(2112K), 0.0126205 secs]
        17552K->15855K(773376K), 0.0127482 secs]
        [Times: user=0.01 sys=0.00, real=0.01 secs]
    [CMS-concurrent-mark: 0.267/0.374 secs]
        [Times: user=4.72 sys=0.01, real=0.37 secs]
    [GC
        [ParNew: 2111K->64K(2112K), 0.0190851 secs]
        17903K->16154K(773376K), 0.0191903 secs]
        [Times: user=0.01 sys=0.00, real=0.02 secs]
    [CMS-concurrent-preclean-start]
    [CMS-concurrent-preclean: 0.044/0.064 secs]
        [Times: user=0.11 sys=0.00, real=0.06 secs]
    [CMS-concurrent-abortable-preclean-start]
    [CMS-concurrent-abortable-clean]  0.031/0.044 secs]
        [Times: user=0.09 sys=0.00, real=0.04 secs]
    [GC
        [YG occupancy: 1515 K (2112K)
        [Rescan (parallel) , 0.0108373 secs]
        [weak refs processing, 0.0000186 secs]
        [1 CMS-remark: 16090K(20288K)]
        17242K(773376K), 0.0210460 secs]
        [Times: user=0.01 sys=0.00, real=0.02 secs]
    [GC
        [ParNew: 2112K->63K(2112K), 0.0716116 secs]
        18177K->17382K(773376K), 0.0718204 secs]
        [Times: user=0.02 sys=0.00, real=0.07 secs]
    [CMS-concurrent-sweep-start]
    [GC
        [ParNew: 2111K->63K(2112K), 0.0830392 secs]
        19363K->18757K(773376K), 0.0832943 secs]
        [Times: user=0.02 sys=0.00, real=0.08 secs]
    [GC
        [ParNew: 2111K->0K(2112K), 0.0035190 secs]
        17527K->15479K(773376K), 0.0036052 secs]
        [Times: user=0.00 sys=0.00, real=0.00 secs]
    [CMS-concurrent-sweep: 0.291/0.662 secs]
        [Times: user=0.28 sys=0.01, real=0.66 secs]
    [GC
        [ParNew: 2048K->0K(2112K), 0.0013347 secs]
        17527K->15479K(773376K), 0.0014231 secs]
        [Times: user=0.00 sys=0.00, real=0.00 secs]
    [CMS-concurrent-reset-start]
    [CMS-concurrent-reset: 0.016/0.016 secs]
        [Times: user=0.01 sys=0.00, real=0.02 secs]
    [GC
        [ParNew: 2048K->1K(2112K), 0.0013936 secs]
        17527K->15479K(773376K), 0.0014814 secs]
        [Times: user=0.00 sys=0.00, real=0.00 secs]

一个CMS周期是由CMS-initial-mark开始到CMS- concurrent-reset结束。  
CMS-concurrent-mark表示标记阶段的结束  
CMS-concurrent-preclean和CMS-concurrent-abortable-preclean表示可以并发的完成再标记工作  
CMS-remark表示再标记阶段  
CMS-concurrent-sweep表示并发清除阶段的结束,这个阶段清除那些被标记为不可达的对象  
    
    
初始标记需要暂停很短的时间然后进入并发阶段。  
重新标记的暂停时间取决于具体的应用情况。（如果对象修改的频繁就需要更久的时间）  
需要重点关心的是CMS-concurrent-sweep-start和CMS-concurrent-sweep这两个阶段的java堆的回收情况  
如果并发清扫阶段只回收了很少的内存，这可能是由：

>   CMS只找到很少的可回收的对象  
>   对象已经进入老年代

无论哪一种情况，JVM都需要调优了

##### -XX:+PrintGCDetails
打印垃圾回收时间,格式为YYYY-MM-DD-T-HH-MM-SS.mmm-TZ

##### -Xloggc
把GC信息写入文件.用法： -Xloggc:&lt;filename&gt;


##### -XX:+PrintGCApplicationConcurrentTime
safepoint操作之间的时间间隔

##### -XX:+PrintGCApplicationStoppedTime
GC所暂停线程的时间

### 锁竞争
jstack 

    ”Read Thread-33” prio=6 tid=0x02b1d400 nid=0x5c0 runnable
    [0x0424f000..0x0424fd94]
        java.lang.Thread.State: RUNNABLE
        at Queue.dequeue(Queue.java:69)
        - locked <0x22e88b10> (a Queue)
        at ReadThread.getWorkItemFromQueue(ReadThread.java:32) 
        at ReadThread.run(ReadThread.java:23)

    ”Writer Thread-29” prio=6 tid=0x02b13c00 nid=0x3cc waiting for monitor
    entry [0x03f7f000..0x03f7fd94]
        java.lang.Thread.State: BLOCKED (on object monitor)
            at Queue.enqueue(Queue.java:31)
            - waiting to lock <0x22e88b10> (a Queue)
            at WriteThread.putWorkItemOnQueue(WriteThread.java:54) 
            at WriteThread.run(WriteThread.java:47)

    ”Writer Thread-26” prio=6 tid=0x02b0d400 nid=0x194 waiting for monitor
    entry [0x03d9f000..0x03d9fc94]
        java.lang.Thread.State: BLOCKED (on object monitor)
            at Queue.enqueue(Queue.java:31)
            - waiting to lock <0x22e88b10> (a Queue)
            at WriteThread.putWorkItemOnQueue(WriteThread.java:54) 
            at WriteThread.run(WriteThread.java:47)

    ”Read Thread-23” prio=6 tid=0x02b08000 nid=0xbf0 waiting for monitor entry
    [0x03c0f000..0x03c0fb14]
        java.lang.Thread.State: BLOCKED (on object monitor)
        at Queue.dequeue(Queue.java:55)
        - waiting to lock <0x22e88b10> (a Queue)
        at ReadThread.getWorkItemFromQueue(ReadThread.java:32) 
        at ReadThread.run(ReadThread.java:23)

Read Thread-33 已获取locked &lt;0x22e88b10&gt;

其它都在waiting to lock &lt;0x22e88b10&gt; (a Queue).
