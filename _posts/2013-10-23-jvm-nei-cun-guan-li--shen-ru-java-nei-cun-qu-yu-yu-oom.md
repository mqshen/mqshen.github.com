---
layout: post
title: "JVM内存管理：深入Java内存区域与OOM"
description: "Java与C++之间有一堵由内存动态分配和垃圾收集技术所围成的高墙，墙外面的人想进去，墙里面的人却想出来。"
category: ""
tags: [JVM调优]
---
{% include JB/setup %}

### 概述：
对于从事C、C++程序开发的开发人员来说，在内存管理领域，他们即是拥有最高权力的皇帝又是执行最基础工作的劳动人民——拥有每一个对象的“所有权”，又担负着每一个对象生命开始到终结的维护责任。  
对于Java程序员来说，不需要在为每一个new操作去写配对的delete/free，不容易出现内容泄漏和内存溢出错误，看起来由JVM管理内存一切都很美好。不过，也正是因为Java程序员把内存控制的权力交给了JVM，一旦出现泄漏和溢出，如果不了解JVM是怎样使用内存的，那排查错误将会是一件非常困难的事情。

### VM运行时数据区域
JVM执行Java程序的过程中，会使用到各种数据区域，这些区域有各自的用途、创建和销毁时间。根据《Java虚拟机规范（第二版）》（下文称VM Spec）的规定，JVM包括下列几个运行时数据区域：

#### 1.程序计数器（Program Counter Register）：
>   每一个Java线程都有一个程序计数器来用于保存程序执行到当前方法的哪一个指令，对于非Native方法，这个区域记录的是正在执行的VM原语的地址，如果正在执行的是Natvie方法，这个区域则为空（undefined）。此内存区域是唯一一个在VM Spec中没有规定任何OutOfMemoryError情况的区域。

#### 2.Java虚拟机栈（Java Virtual Machine Stacks）
>   与程序计数器一样，VM栈的生命周期也是与线程相同。VM栈描述的是Java方法调用的内存模型：每个方法被执行的时候，都会同时创建一个帧（Frame）用于存储本地变量表、操作栈、动态链接、方法出入口等信息。每一个方法的调用至完成，就意味着一个帧在VM栈中的入栈至出栈的过程。在后文中，我们将着重讨论VM栈中本地变量表部分。
>   经常有人把Java内存简单的区分为堆内存（Heap）和栈内存（Stack），实际中的区域远比这种观点复杂，这样划分只是说明与变量定义密切相关的内存区域是这两块。其中所指的“堆”后面会专门描述，而所指的“栈”就是VM栈中各个帧的本地变量表部分。本地变量表存放了编译期可知的各种标量类型（boolean、byte、char、short、int、float、long、double）、对象引用（不是对象本身，仅仅是一个引用指针）、方法返回地址等。其中long和double会占用2个本地变量空间（32bit），其余占用1个。本地变量表在进入方法时进行分配，当进入一个方法时，这个方法需要在帧中分配多大的本地变量是一件完全确定的事情，在方法运行期间不改变本地变量表的大小。
>   在VM Spec中对这个区域规定了2中异常状况：如果线程请求的栈深度大于虚拟机所允许的深度，将抛出StackOverflowError异常；如果VM栈可以动态扩展（VM Spec中允许固定长度的VM栈），当扩展时无法申请到足够内存则抛出OutOfMemoryError异常。

#### 3.本地方法栈（Native Method Stacks）
>   本地方法栈与VM栈所发挥作用是类似的，只不过VM栈为虚拟机运行VM原语服务，而本地方法栈是为虚拟机使用到的Native方法服务。它的实现的语言、方式与结构并没有强制规定，甚至有的虚拟机（譬如Sun Hotspot虚拟机）直接就把本地方法栈和VM栈合二为一。和VM栈一样，这个区域也会抛出StackOverflowError和OutOfMemoryError异常。

#### 4.Java堆（Java Heap）
>   对于绝大多数应用来说，Java堆是虚拟机管理最大的一块内存。Java堆是被所有线程共享的，在虚拟机启动时创建。Java堆的唯一目的就是存放对象实例，绝大部分的对象实例都在这里分配。这一点在VM Spec中的描述是：所有的实例以及数组都在堆上分配（原文：The heap is the runtime data area from which memory for all class instances and arrays is allocated），但是在逃逸分析和标量替换优化技术出现后，VM Spec的描述就显得并不那么准确了。
>   Java堆内还有更细致的划分：新生代、老年代，再细致一点的：eden、from survivor、to survivor，甚至更细粒度的本地线程分配缓冲（TLAB）等，无论对Java堆如何划分，目的都是为了更好的回收内存，或者更快的分配内存，在本章中我们仅仅针对内存区域的作用进行讨论，Java堆中的上述各个区域的细节，可参见本文第二章《JVM内存管理：深入垃圾收集器与内存分配策略》。
>   根据VM Spec的要求，Java堆可以处于物理上不连续的内存空间，它逻辑上是连续的即可，就像我们的磁盘空间一样。实现时可以选择实现成固定大小的，也可以是可扩展的，不过当前所有商业的虚拟机都是按照可扩展来实现的（通过-Xmx和-Xms控制）。如果在堆中无法分配内存，并且堆也无法再扩展时，将会抛出OutOfMemoryError异常。

#### 5.方法区（Method Area）
>   叫“方法区”可能认识它的人还不太多，如果叫永久代（Permanent Generation）它的粉丝也许就多了。它还有个别名叫做Non-Heap（非堆），但是VM Spec上则描述方法区为堆的一个逻辑部分（原文：the method area is logically part of the heap），这个名字的问题还真容易令人产生误解，我们在这里就不纠结了。
>   方法区中存放了每个Class的结构信息，包括常量池、字段描述、方法描述等等。VM Space描述中对这个区域的限制非常宽松，除了和Java堆一样不需要连续的内存，也可以选择固定大小或者可扩展外，甚至可以选择不实现垃圾收集。相对来说，垃圾收集行为在这个区域是相对比较少发生的，但并不是某些描述那样永久代不会发生GC（至少对当前主流的商业JVM实现来说是如此），这里的GC主要是对常量池的回收和对类的卸载，虽然回收的“成绩”一般也比较差强人意，尤其是类卸载，条件相当苛刻。

#### 6.运行时常量池（Runtime Constant Pool）
>   Class文件中除了有类的版本、字段、方法、接口等描述等信息外，还有一项信息是常量表(constant_pool table)，用于存放编译期已可知的常量，这部分内容将在类加载后进入方法区（永久代）存放。但是Java语言并不要求常量一定只有编译期预置入Class的常量表的内容才能进入方法区常量池，运行期间也可将新内容放入常量池（最典型的String.intern()方法.Jave7开始就放在Heap中了）。
>   运行时常量池是方法区的一部分，自然受到方法区内存的限制，当常量池无法在申请到内存时会抛出OutOfMemoryError异常。

#### 7.本机直接内存（Direct Memory）
>   直接内存并不是虚拟机运行时数据区的一部分，它根本就是本机内存而不是VM直接管理的区域。但是这部分内存也会导致OutOfMemoryError异常出现，因此我们放到这里一起描述。
>   在JDK1.4中新加入了NIO类，引入一种基于渠道与缓冲区的I/O方式，它可以通过本机Native函数库直接分配本机内存，然后通过一个存储在Java堆里面的DirectByteBuffer对象作为这块内存的引用进行操作。这样能在一些场景中显著提高性能，因为避免了在Java对和本机堆中来回复制数据。
>   显然本机直接内存的分配不会受到Java堆大小的限制，但是即然是内存那肯定还是要受到本机物理内存（包括SWAP区或者Windows虚拟内存）的限制的，一般服务器管理员配置JVM参数时，会根据实际内存设置-Xmx等参数信息，但经常忽略掉直接内存，使得各个内存区域总和大于物理内存限制（包括物理的和操作系统级的限制），而导致动态扩展时出现OutOfMemoryError异常。

### 实战OutOfMemoryError
上述区域中，除了程序计数器，其他在VM Spec中都描述了产生OutOfMemoryError（下称OOM）的情形，那我们就实战模拟一下，通过几段简单的代码，令对应的区域产生OOM异常以便加深认识，同时初步介绍一些与内存相关的虚拟机参数。  
下文的代码都是基于

    java version "1.7.0_40"
    Java(TM) SE Runtime Environment (build 1.7.0_40-b43)
    Java HotSpot(TM) 64-Bit Server VM (build 24.0-b56, mixed mode)

#### Java堆
Java堆存放的是对象实例，因此只要不断建立对象，并且保证GC Roots到对象之间有可达路径即可产生OOM异常。测试中限制Java堆大小为20M，不可扩展，通过参数-XX:+HeapDumpOnOutOfMemoryError让虚拟机在出现OOM异常的时候Dump出内存映像以便分析。（关于Dump映像文件分析方面的内容，可参见本文第三章《JVM内存管理：深入JVM内存异常分析与调优》。）

清单1：Java堆OOM测试

    package org.goldratio.memory;
    
    import java.util.ArrayList;
    import java.util.List;
    
    /**
    * VM Args：-Xms20m -Xmx20m -XX:+HeapDumpOnOutOfMemoryError
    */
    public class HeapOOM {
    
          static class OOMObject {
          }
    
          public static void main(String[] args) {
                 List<OOMObject> list = new ArrayList<OOMObject>();
    
                 while (true) {
                        list.add(new OOMObject());
                 }
          }
    }

运行结果：

    java.lang.OutOfMemoryError: Java heap space
    Dumping heap to java_pid1276.hprof ...
    Heap dump file created [27561240 bytes in 0.180 secs]

#### VM栈和本地方法栈
Hotspot虚拟机并不区分VM栈和本地方法栈，因此-Xoss参数实际上是无效的，栈容量只由-Xss参数设定。关于VM栈和本地方法栈在VM Spec描述了两种异常：StackOverflowError与OutOfMemoryError，当栈空间无法继续分配分配时，到底是内存太小还是栈太大其实某种意义上是对同一件事情的两种描述而已，在实验中，对于单线程应用尝试下面2种方法均无法让虚拟机产生OOM，全部尝试结果都是获得SOF异常。
1. 使用-Xss参数削减栈内存容量。结果：抛出SOF异常时的堆栈深度相应缩小。
2. 定义大量的本地变量，增大此方法对应帧的长度。结果：抛出SOF异常时的堆栈深度相应缩小。
<!--
3. 创建几个定义很多本地变量的复杂对象，打开逃逸分析和标量替换选项，使得JIT编译器允许对象拆分后在栈中分配。结果：实际效果同第二点。
-->

清单2：VM栈和本地方法栈OOM测试（仅作为第1点测试程序）

    package org.goldratio.memory;
    
    /**
     * VM Args：-Xss256k
     */
    public class JavaVMStackSOF {
     
           private int stackLength = 1;
     
           public void stackLeak() {
                  stackLength++;
                  stackLeak();
           }
     
           public static void main(String[] args) throws Throwable {
                  JavaVMStackSOF oom = new JavaVMStackSOF();
                  try {
                         oom.stackLeak();
                  } catch (Throwable e) {
                         System.out.println("stack length:" + oom.stackLength);
                         throw e;
                  }
           }
    }

运行结果：

    stack length:1891Exception in thread "main" 
    java.lang.StackOverflowError
    	at org.goldratio.memory.JavaVMStackSOF.stackLeak(JavaVMStackSOF.java:11)

如果在多线程环境下，不断建立线程倒是可以产生OOM异常。原因其实很好理解，操作系统分配给每个进程的内存是有限制的，譬如32位Windows限制为2G，Java堆和方法区的大小JVM有参数可以限制最大值，那剩余的内存为2G（操作系统限制）-Xmx（最大堆）-MaxPermSize（最大方法区），程序计数器消耗内存很小，可以忽略掉，那虚拟机进程本身耗费的内存不计算的话，剩下的内存就供每一个线程的VM栈和本地方法栈瓜分了，那自然每个线程中VM栈分配内存越多，就越容易把剩下的内存耗尽。

#### 运行时常量池
要在常量池里添加内容，最简单的就是使用String.intern()这个Native方法。由于常量池分配在方法区内，我们只需要通过-XX:PermSize和-XX:MaxPermSize限制方法区大小即可限制常量池容量。实现代码如下：
清单4：运行时常量池导致的OOM异常

    package org.goldratio.memory;
    
    import java.util.ArrayList;
    import java.util.List;
    
    /**
     * VM Args：-XX:PermSize=4M -XX:MaxPermSize=4M
     */
    public class RuntimeConstantPoolOOM {
    
    	public static void main(String[] args) {
    		// 使用List保持着常量池引用，压制Full GC回收常量池行为
    		List<String> list = new ArrayList<String>();
    		// 10M的PermSize在integer范围内足够产生OOM了
    		int i = 0;
    		while (true) {
    			list.add(String.valueOf(i++).intern());
    		}
    	}
    }

未出现异常，监控jvm看到Heap一直在增长。这是因为Java7 String.intern 被分配再了Heap中参见[Java SE 7 ](http://www.oracle.com/technetwork/java/javase/jdk7-relnotes-418459.html)

#### 方法区
上文讲过，方法区用于存放Class相关信息，所以这个区域的测试我们借助CGLib直接操作字节码动态生成大量的Class，值得注意的是，这里我们这个例子中模拟的场景其实经常会在实际应用中出现：当前很多主流框架，如Spring、Hibernate对类进行增强时，都会使用到CGLib这类字节码技术，当增强的类越多，就需要越大的方法区用于保证动态生成的Class可以加载入内存。
清单5：借助CGLib使得方法区出现OOM异常

    package org.goldratio.memory;
    
    import java.lang.reflect.Method;
    
    import net.sf.cglib.proxy.Enhancer;
    import net.sf.cglib.proxy.MethodInterceptor;
    import net.sf.cglib.proxy.MethodProxy;
    
    
    /**
     * VM Args： -XX:PermSize=10M -XX:MaxPermSize=10M
     * 
     */
    public class JavaMethodAreaOOM {
    
    	public static void main(String[] args) {
    		while (true) {
    			Enhancer enhancer = new Enhancer();
    			enhancer.setSuperclass(OOMObject.class);
    			enhancer.setUseCache(false);
    			enhancer.setCallback(new MethodInterceptor() {
    
    				@Override
    				public Object intercept(Object obj, Method method,
    						Object[] args, MethodProxy proxy) throws Throwable {
    					return proxy.invokeSuper(obj, args);
    				}
    			});
    			enhancer.create();
    		}
    	}
    
    	static class OOMObject {
    
    	}
    }

运行结果：

    Exception in thread "main" 
    Exception: java.lang.OutOfMemoryError thrown from the UncaughtExceptionHandler in thread "main"
    
#### 本机直接内存
DirectMemory容量可通过-XX:MaxDirectMemorySize指定，不指定的话默认与Java堆（-Xmx指定）一样，下文代码越过了DirectByteBuffer，直接通过反射获取Unsafe实例进行内存分配（Unsafe类的getUnsafe()方法限制了只有引导类加载器才会返回实例，也就是基本上只有rt.jar里面的类的才能使用），因为DirectByteBuffer也会抛OOM异常，但抛出异常时实际上并没有真正向操作系统申请分配内存，而是通过计算得知无法分配既会抛出，真正申请分配的方法是unsafe.allocateMemory()。

### 总结
到此为止，我们弄清楚虚拟机里面的内存是如何划分的，哪部分区域，什么样的代码、操作可能导致OOM异常。虽然Java有垃圾收集机制，但OOM仍然离我们并不遥远，本章内容我们只是知道各个区域OOM异常出现的原因，下一章我们将看看Java垃圾收集机制为了避免OOM异常出现，做出了什么样的努力。
