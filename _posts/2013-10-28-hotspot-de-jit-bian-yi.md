---
layout: post
title: "HotSpot的JIT编译"
description: "Java是一种“动态编译”语言。也就是说在程序运行时,其中的类还会再进行一次编译,然后转换成机器码。这个过程称为即时编译或JITing,并且通常是一次处理一个方法。要在庞大的代码库中找出其中的重要部分,理解这个过程是关键。"
category: ""
tags: [HotSpot, JIT, Java]
---
{% include JB/setup %}

Java是一种“动态编译”语言。也就是说在程序运行时,其中的类 还会再进行一次编译,然后转换成机器码。
这个过程称为即时编译或JITing,并且通常是一次处理一个方法。要在庞大的代码库中找出其中的重要部分,理解这个过程是关键。

>    几乎所有现代JVM中都有某种JIT编译器
>    相比较而言,纯粹解释型的VM要慢得多
>    编译过的方法在运行速度上要比解释型的代码快很多,非常多
>    先编译用得最多的方法,这是有道理的
>    在做JIT编译时,先处理唾手可得的编译很重要

方法一开始都是以字节码形态存在的,有调用时JVM只会对字节码进行解释并执行,同时记 录方法被调用的次数及其他一些统计数据。当被调用次数达到某个阈值(默认10 000次)后,如果它是合格的方法,就会有个JVM线程在后台把它的字节码编译成机器码。如果编译成功,以后 所有对该方法的调用都会用它的编译结果,除非出现了某些导致检验失效的情况,或者出现了逆优化.   
根据实际情况,方法编译后产生的机器码运行速度可能比解释模式下的字节码快100倍。改善性能通常都要先弄明白程序中哪些方法比较重要,以及哪些重要的方法被编译了。

### 介绍HotSpot
Oracle收购Sun时拿到了HotSpot VM(原来收购BEA时还拿到一个JRockit)。HotSpot是 OpenJDK的基础。它有两种运行模式——客户端模式和服务器端模式。可以在启动JVM时指定 -client或-server选项来选择不同的模式。(必须是命令行中的第一个选项。)每种模式都有各自适用的应用程序。

##### 客户端编译器
客户端编译器主要用于GUI应用程序。在这个领域中,操作的一致性至关重要,所以客户端 编译器(有时叫C1)在编译时所做的决定往往更保守。也就是说它不能因为要取消一个经证实不 正确或基于错误假设的优化决定而意外暂停。

##### 服务器端编译器
相反,服务器端编译器(C2)在编译时会大胆假设。为了确保代码正确运行,C2会快速地做一次运行时检查(通常被称为警戒条件),以确保假设有效。如果假设无效,它会取消这次编 译,并尝试别的编译。这种大胆假设的方式比保守的客户端编译器产生的编译结果性能好很多。

##### 实时Java
近年来出现了一种实时Java平台,有些开发人员好奇为什么那些需要表现出高性能的代码不 直接用这个平台(它是独立的JVM,不是HotSpot选件)。那是因为实时系统不一定是最快的。  
实时编程的关注点实际上是承诺能否兑现。从统计角度讲,实时系统是为了让执行操作的时 间尽量保持一致,并且为了达成这个目的,它可能会牺牲一些平均等待时间。为了让运行状况保 持一致,整体性能是可以受到轻微影响的。  
但希望实现高性能表现的团队想要的是更低的平均等待时间,即便以更高的方差为代价,所 以他们通常会选择服务器端编译器的大胆优化策略.

### 内联方法
内联是HotSpot的最大卖点之一。内联的方法不再是被调用,而是将调用方法的代码直接放到调用者内部。  
平台有这方面的优势,编译器可以根据运行时的统计数据(方法的调用频率)和其他因素(比如会不会因为调用者方法太多而对代码缓存产生影响)来决定如何处理内联。也就是说HotSpot编译器所做的内联决策比提前编译的编译器更智能。    
方法的内联是完全自动的,并且默认参数值几乎适用于任何情况。但也有选项可以用来控制 内联方法大小,以及方法在成为内联候选之前的调用频率要达到多高。对于好奇的程序员来说, 这些选项对于深入了解内联如何工作很有帮助。通常它们对于生产环境下的代码用处不大,并且 应该作为性能调优的最后选择,因为它们对运行时系统的性能可能存在不可预测的影响。

### 动态编译和独占调用
独占调用就是这种大胆优化的例子之一。它是基于大量观察做出的优化,像下面这种对象上的方法调用:


{% highlight java linenos %}
    MyActualClassNotInterface obj = getInstance();
    obj.callMyMethod();
{% endhighlight %}

只会在一种类型的对象上调用。换句话说,就是调用点obj.callMyMethod()几乎不会同时碰到一个类和它的子类。这时可以把Java方法查找替换为callMyMethod()编译结果的直接调用。

### 读懂编译日志
使用

    -XX:+PrintCompilation

告诉JIT编译线程输出标准日志.这些日志表明方法超过编译阈值并被转成机器 码的时间。

    230    1             java.lang.String::hashCode (55 bytes)
    243    2             java.lang.String::indexOf (70 bytes)
    251    3             sun.nio.cs.UTF_8$Encoder::encode (361 bytes)
    251    4             java.lang.String::charAt (29 bytes)
    322    5             java.lang.String::equals (81 bytes)
    330    6             java.lang.Object::<init> (1 bytes)
    356    7             java.lang.String::length (6 bytes)
    382    8             java.lang.String::lastIndexOf (52 bytes)
    411    9             java.util.Properties$LineReader::readLine (452 bytes)
    414   10             java.util.Properties::loadConvert (505 bytes)
    420   13     n       java.lang.System::arraycopy (0 bytes)   (static)
    428   11             java.lang.String::indexOf (166 bytes)
    429   12             java.io.UnixFileSystem::normalize (75 bytes)
    435   14             java.lang.Math::min (11 bytes)
    435   15             java.lang.String::replace (127 bytes)
    511   16             java.lang.String::startsWith (72 bytes)
    562   17             java.util.Arrays::fill (28 bytes)
    604   18             java.util.Properties::load0 (250 bytes)
    608   19             sun.net.www.ParseUtil::encodePath (336 bytes)
   1185    1 %           com.sun.org.apache.xerces.internal.impl.io.UTF8Reader::read @ 113 (1388 bytes)
   1694   20             com.sun.org.apache.xerces.internal.impl.io.UTF8Reader::read (1388 bytes)
   1700   21             sun.reflect.ClassFileAssembler::emitByte (11 bytes)
   1702   22             sun.reflect.ByteVectorImpl::add (38 bytes)
   1711   23             java.lang.reflect.Method::getName (5 bytes)
   1712   24             java.lang.reflect.Method::getReturnType (5 bytes)
   1712   25             java.lang.reflect.Method::getDeclaringClass (5 bytes)
   1729   26             com.sun.org.apache.xerces.internal.util.XMLChar::isContent (35 bytes)
   1731   27             com.sun.org.apache.xerces.internal.impl.XMLEntityScanner::scanLiteral (840 bytes)

这是非常典型的PrintCompilation输出。这些日志表明了“热”到可以编译的方法。这些输出中每行都有个数字,表明了这些方法在这次运行中的编译顺序。注意,由于平台的 动态性质,这个顺序在每次运行时可能会稍有变化。这里还有一些其他域。
>   s   表明该方法是同步的
>   !   表明方法有异常处理。
>   %   当前栈替换(OSR)。这个方法被编译了,并且换掉了运行代码中的解释型版本。
注意,OSR方法有它们自己的计数方案,从1开始。

##### 小心僵尸
当查看用服务器端编译器(C2)运行代码的样例日志时,你可能偶尔会看到"made not entrant" 或"made zombie"这样的字眼。这表明由于类加载操作(通常情况下),某个已经被编译过的特定 方法现在无效了。
##### 逆优化
如果经证实代码优化所基于的假设是不真实的,HotSpot可以对代码进行逆优化操作。在许 多情况下,它会重新考虑,尝试不同的优化。因此同一个方法可能会被逆优化和重编译几次。

