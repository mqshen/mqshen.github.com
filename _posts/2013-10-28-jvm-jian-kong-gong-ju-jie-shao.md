---
layout: post
title: "JVM监控工具介绍"
description: "JVM监控工具介绍 jps jmap"
category: ""
tags: [Java performance]
---
{% include JB/setup %}

### jps
列出所有的jvm实例

{% highlight sh linenos %}
    $ jps
    13877 Jps
    10019 Bootstrap
    4364 activemq.jar
{% endhighlight %}

{% highlight sh linenos %}
    $ jps host
{% endhighlight %}
列出远程服务器上的JVM实例,采用RMI协议，默认连接端口为1099

### jmap
用来显示Java进程的内存映射(也能分析Java core文件)

#### 默认视图
jmap最简单的用法是查看连接到进程里的本地类库。除非你的应用程序里有很多JNI代码, 否则这种用法通常没什么用。
{% highlight sh linenos %}
$ jmap 10019
    Attaching to process ID 10019, please wait...
    Debugger attached successfully.
    Server compiler detected.
    JVM version is 23.25-b01
    0x0000000000400000	7K	/home/microloan/software/jdk1.7.0_25/bin/java
    0x00000034c3e00000	153K	/lib64/ld-2.12.so
    0x00000034c4200000	1877K	/lib64/libc-2.12.so
    0x00000034c4600000	584K	/lib64/libm-2.12.so
    0x00000034c4a00000	22K	/lib64/libdl-2.12.so
    0x00000034c4e00000	142K	/lib64/libpthread-2.12.so
    0x00000034c5600000	45K	/lib64/librt-2.12.so
    0x00000034c6200000	111K	/lib64/libresolv-2.12.so
    0x00000034ce600000	91K	/lib64/libgcc_s-4.4.6-20110824.so.1
    0x00007f7880f42000	474K	/home/microloan/software/jdk1.7.0_25/jre/lib/amd64/libt2k.so
    0x00007f7881934000	490K	/home/microloan/software/jdk1.7.0_25/jre/lib/amd64/libfontmanager.so
    0x00007f78822b1000	26K	/lib64/libnss_dns-2.12.so
    0x00007f78884c1000	250K	/home/microloan/software/jdk1.7.0_25/jre/lib/amd64/libsunec.so
    0x00007f7888708000	36K	/home/microloan/software/jdk1.7.0_25/jre/lib/amd64/headless/libmawt.so
    0x00007f788890f000	759K	/home/microloan/software/jdk1.7.0_25/jre/lib/amd64/libawt.so
    0x00007f7888be2000	44K	/home/microloan/software/jdk1.7.0_25/jre/lib/amd64/libmanagement.so
    0x00007f7888dea000	107K	/home/microloan/software/jdk1.7.0_25/jre/lib/amd64/libnet.so
    0x00007f78901e3000	89K	/home/microloan/software/jdk1.7.0_25/jre/lib/amd64/libnio.so
    0x00007f789a4de000	120K	/home/microloan/software/jdk1.7.0_25/jre/lib/amd64/libzip.so
    0x00007f789a6f9000	64K	/lib64/libnss_files-2.12.so
    0x00007f789a91f000	214K	/home/microloan/software/jdk1.7.0_25/jre/lib/amd64/libjava.so
    0x00007f789ab4a000	63K	/home/microloan/software/jdk1.7.0_25/jre/lib/amd64/libverify.so
    0x00007f789ae59000	13190K	/home/microloan/software/jdk1.7.0_25/jre/lib/amd64/server/libjvm.so
    0x00007f789bb7c000	103K	/home/microloan/software/jdk1.7.0_25/jre/lib/amd64/jli/libjli.so
{% endhighlight %}

#### 堆视图
使用-heap选项时,jmap会抓取进程当前的堆快照。在输出结果中能看到构成Java进程堆内存的基本参数。

{% highlight sh linenos %}
$ jmap -heap 10019
    Attaching to process ID 10019, please wait...
    Debugger attached successfully.
    Server compiler detected.
    JVM version is 23.25-b01
    
    using thread-local object allocation.
    Parallel GC with 4 thread(s)
    
    Heap Configuration:
       MinHeapFreeRatio = 40
       MaxHeapFreeRatio = 70
       MaxHeapSize      = 1073741824 (1024.0MB)
       NewSize          = 1310720 (1.25MB)
       MaxNewSize       = 17592186044415 MB
       OldSize          = 5439488 (5.1875MB)
       NewRatio         = 2
       SurvivorRatio    = 8
       PermSize         = 536870912 (512.0MB)
       MaxPermSize      = 536870912 (512.0MB)
       G1HeapRegionSize = 0 (0.0MB)
    
    Heap Usage:
    PS Young Generation
    Eden Space:
       capacity = 350093312 (333.875MB)
       used     = 274149616 (261.44944763183594MB)
       free     = 75943696 (72.42555236816406MB)
       78.30758446479548% used
    From Space:
       capacity = 3801088 (3.625MB)
       used     = 98304 (0.09375MB)
       free     = 3702784 (3.53125MB)
       2.586206896551724% used
    To Space:
       capacity = 3604480 (3.4375MB)
       used     = 0 (0.0MB)
       free     = 3604480 (3.4375MB)
       0.0% used
    PS Old Generation
       capacity = 715849728 (682.6875MB)
       used     = 209051048 (199.36661529541016MB)
       free     = 506798680 (483.32088470458984MB)
       29.20320282638984% used
    PS Perm Generation
       capacity = 536870912 (512.0MB)
       used     = 141163656 (134.62415313720703MB)
       free     = 395707256 (377.37584686279297MB)
       26.29377990961075% used
    
    36088 interned Strings occupying 3954552 bytes.
{% endhighlight %}

#### 柱状视图
柱状视图显示了系统中每个类型的实例(还有一些内部实体)占用的内存量。各个类型按使 用内存多少排列,这样就比较容易看到最大的内存。  
当然,如果所有内存都交给了框架和平台类,这里可能就没你什么事了。但如果真有一个你 的类,有了这些信息便能更好地干预它的内存占用。

$ jmap -histo 10019|head -30

{% highlight sh linenos %}
     num     #instances         #bytes  class name
    ----------------------------------------------
       1:        228765      175874592  [I
       2:        508052      129038624  [B
       3:        707707       71626144  [C
       4:        245171       37640784  <constMethodKlass>
       5:        245171       33355304  <methodKlass>
       6:         22498       25107264  <constantPoolKlass>
       7:         22498       17390320  <instanceKlassKlass>
       8:        651709       15641016  java.lang.String
       9:         18016       13946432  <constantPoolCacheKlass>
      10:         88801        7104080  java.lang.reflect.Method
      11:         74206        6517832  [Ljava.util.HashMap$Entry;
      12:        197260        6312320  java.util.HashMap$Entry
      13:        121133        4845320  java.util.LinkedHashMap$Entry
      14:        109182        4367280  java.util.HashMap$ValueIterator
      15:        159152        4024816  [Ljava.lang.String;
      16:         69208        3680112  [Ljava.lang.Object;
      17:         44330        3191760  java.util.regex.Pattern
      18:          5365        3107120  <methodDataKlass>
      19:         46070        2948480  java.util.regex.Matcher
      20:         24025        2875640  java.lang.Class
      21:         49420        2767520  java.util.HashMap
      22:        112061        2689464  java.io.File
      23:        109185        2620440  org.apache.catalina.LifecycleEvent
      24:         44025        2465400  [Ljava.util.regex.Pattern$GroupHead;
      25:        109184        2211856  [Lorg.apache.catalina.Container;
      26:         64608        2067456  java.util.concurrent.ConcurrentHashMap$HashEntry
      27:         39373        1961768  [[I
{% endhighlight %}
柱状图模式下还能看到其他有意思的事情,这次指定-histo:live选项。这是告诉jmap只 处理存活对象,而不是整个堆(jmap默认会处理所有对象,也包括还没被收集的垃圾)。让我们 看看这次输出什么:


{% highlight sh linenos %}
    $ jmap -histo:live 10019|head -30
    
     num     #instances         #bytes  class name
    ----------------------------------------------
       1:         74512       47870936  [B
       2:        244687       37588560  <constMethodKlass>
       3:        244687       33289480  <methodKlass>
       4:        241228       31651544  [C
       5:         22264       24880992  <constantPoolKlass>
       6:         22264       17239360  <instanceKlassKlass>
       7:         17782       13773984  <constantPoolCacheKlass>
       8:        237034        5688816  java.lang.String
       9:        171257        5480224  java.util.HashMap$Entry
      10:         64024        5121920  java.lang.reflect.Method
      11:         51623        4688640  [Ljava.util.HashMap$Entry;
      12:          5364        3106664  <methodDataKlass>
      13:         23791        2849368  java.lang.Class
      14:         52890        2774632  [Ljava.lang.Object;
      15:         66549        2661960  java.util.LinkedHashMap$Entry
      16:         60908        1949056  java.util.concurrent.ConcurrentHashMap$HashEntry
      17:         37817        1892168  [[I
      18:         30760        1882104  [S
      19:         28266        1710304  [I
      20:         28325        1586200  java.util.HashMap
      21:         56713        1550616  [Ljava.lang.String;
      22:         19744        1263616  java.util.LinkedHashMap
      23:         24731        1187088  org.apache.catalina.loader.ResourceEntry
      24:         16361        1047104  java.net.URL
      25:         37258         894192  java.util.ArrayList
      26:         22105         884200  java.lang.ref.SoftReference
      27:          1510         869760  <objArrayKlassKlass>
{% endhighlight %}
注意输出的变化查看那些类是消耗内存的主要力量。

使用jmap时应该稍微谨慎点。进行该操作时JVM还在运行(如果你不走运,还有可能在读 取快照期间做了垃圾回收),所以你应该多运行几次,特别是在你看到任何奇怪或太好的结果时。

#### 产生离线导出文件

{% highlight sh linenos %}
    $ jmap -dump:live,format=b,file=heap.hprof 10019
{% endhighlight %}
导出结果可以用来做离线分析,可以留给jmap以后自己用,也可以留给Oracle的jhat等工具做高级分析.

### VisualVM

VisualVM是Oracle JVM自带的可视化工具。它是插件架构,采用标准配置,比JConsole用起来更方便。

{% highlight sh linenos %}
    $jvisualvm
{% endhighlight %}
如图所示:
![VisualVM]({{ BASE_PATH }}/assets/images/jvisualvm.png)
