---
layout: post
title: "JVM内存管理"
description: "Java与C++之间有一堵由内存动态分配和垃圾收集技术所围成的高墙，墙外面的人想进去，墙里面的人却想出来。"
tags: [JVM调优]
---
{% include JB/setup %}
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

未出现异常，监控jvm看到Heap一直在增长。这是因为Java7 String.intern 被分配再了Heap中.

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


#### 参考资料
>   [高级语言虚拟机知识库-JVM基础](http://hllvm.group.iteye.com/group/wiki?category_id=316)  
>   [Java Performance](http://book.douban.com/subject/5980062/)  
>   [Java SE 7 ](http://www.oracle.com/technetwork/java/javase/jdk7-relnotes-418459.html)

