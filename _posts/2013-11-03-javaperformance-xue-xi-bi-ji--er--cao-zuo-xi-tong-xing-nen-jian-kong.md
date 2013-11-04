---
layout: post
title: "操作系统性能监控"
description: "Linux操作系统性能监控的工具介绍"
category: ""
tags: [performance, linux]
---
{% include JB/setup %}

### CPU利用率
##### top

>   对多核它会显示多个CPU所占百分比的总和。
>   进入视图后按1就进入按核显示视图
>   shift+h就可以进入按线程显示视图

##### vmstat
{% highlight sh linenos %}

$ vmstat 5
procs --------------memory------------ ---swap-- -----io---- --system-- -----cpu-----
 r  b   swpd   free     buff    cache   si   so    bi    bo   in   cs   us sy id wa st
 0  0 14892532 121444   25072   57084   45   18    59    41    0    0   2  0  95  2  0
 0  0 14876388 135656   24376   57860   45   18    59    41    0    0   2  0  95  2  0	
 0  0 14876308 135516   24384   57808  146    0   168    14 1562 2132   1  0  98  1  0	
 0  0 14876268 135268   24384   57936   56    0    56     0 1469 2042   0  0  99  0  0	
 0  0 14876248 134624   24392   57888   54    0    54    48 1678 2144   1  1  98  0  0	
 0  0 14876204 134524   24408   57948  108    0   108    14 1452 2077   0  0  99  1  0	
 0  0 14876184 134400   24408   57908   48    0    48     0 1495 2020   1  0  99  0  0	
 0  0 14876092 134488   24416   57856  150    0   150    18 1490 2093   1  0  98  1  0	
 0  0 14876048 134372   24416   57916   62    0    62     0 1442 1988   0  0  99  0  0	
 0  0 14876008 133876   24432   57892   84    0    84   122 1534 2075   1  0  99  1  0	
 0  0 14875896 133504   24464   57792  278    0   278    74 1488 2078   0  0  97  2  0

{% endhighlight %}
vmstat 参数为获取信息的时间间隔

关注CPU那一列
us 表示用户CPU利用率
sy 表示系统CPU利用率
id 表示CPU的空闲率


### CPU调度运行队列
##### vmstat
{% highlight sh linenos %}

$ vmstat
procs --------------memory------------ ---swap-- -----io---- --system-- -----cpu-----
 r  b   swpd   free     buff    cache   si   so    bi    bo   in   cs   us sy id wa st
 0  0 14892532 121444   25072   57084   45   18    59    41    0    0   2  0  95  2  0

{% endhighlight %}
系统运行队列深度由procs列的r表示


### 内存利用率
##### vmstat
{% highlight sh linenos %}

$ vmstat
procs --------------memory------------ ---swap-- -----io---- --system-- -----cpu-----
 r  b   swpd   free     buff    cache   si   so    bi    bo   in   cs   us sy id wa st
 0  0 14892532 121444   25072   57084   45   18    59    41    0    0   2  0  95  2  0

{% endhighlight %}
memory列下的free表示空闲内存
swap列下的si so 表示 page-in 和page-out的数量

### 锁争用
##### pidstat
{% highlight sh linenos %}
# pidstat -w -I -p 11284 5
Linux 2.6.32-220.el6.x86_64 (belinkdev) 	2013年11月04日 	_x86_64_	(4 CPU)

19时17分32秒       PID   cswch/s nvcswch/s  Command
19时17分37秒     11284      0.00      0.00  java
19时17分42秒     11284      0.00      0.00  java
19时17分47秒     11284      0.00      0.00  java
19时17分52秒     11284      0.00      0.00  java
19时17分57秒     11284      0.00      0.00  java
19时18分02秒     11284      0.00      0.00  java

{% endhighlight %}

cswch/s表示每秒voluntary context switches的数量
nvcswch/s表示每秒involuntary context switches的数量
involuntary context switches一般是由线程抢占所引起的

百分比=切换数量/虚拟处理器数量 * 上下文切换所需要的时钟周期 / 处理器频率
当该值达到3% to 5%就应该进行相关的排查工作。  
每个上下文交换需要80000以上个clock cycle。  

### 线程迁移


### 网络I/O
netstat -i
[nicstat](http://sourceforge.net/projects/nicstat/files/)需要自己安装
安装时需要修改Makefile

{% highlight sh linenos %}
CFLAGS =	$(COPT) -m32		        #将此行修改为如下：
CFLAGS =    $(COPT)
{% endhighlight %}

{% highlight sh linenos %}
# nicstat 
    Time      Int   rKB/s   wKB/s   rPk/s   wPk/s    rAvs    wAvs %Util    Sat
19:29:48       lo   18.13   18.13   54.81   54.81   338.7   338.7  0.00   0.00
19:29:48      em1    3.02    2.95   27.37    3.45   112.9   877.4  0.00   0.00
19:29:48   vmnet1    0.00    0.00    0.00    0.00    0.00    0.00  0.00   0.00
19:29:48   vmnet8    0.00    0.00    0.00    0.00    0.00    0.00  0.00   0.00
19:29:48   virbr0    0.00    0.00    0.00    0.01    0.00   47.24  0.00   0.00
{% endhighlight %}

Int     网卡名字
rKB/s   每秒读取的数量
wKB/s   每秒写的数量
rPk/s   每秒读取包的数量
wPk/s   每秒写包的数量
rAvs    平均每次读取读取的字节数
wAvs    平均每次写写的字节数
%Util   网卡利用率(一秒中有百分之多少的时间用于网络I/0操作)
Sat     网卡的饱和度



### 硬盘I/O
iostat
{% highlight sh linenos %}
# iostat -d -k 1 10
Linux 2.6.32-220.el6.x86_64 (belinkdev) 	2013年11月04日 	_x86_64_	(4 CPU)

Device:            tps    kB_read/s    kB_wrtn/s    kB_read    kB_wrtn
sda              26.32       236.35       162.76 2949548902 2031207616
dm-0              7.90        24.85        24.72  310152713  308489884
dm-1             63.10       180.71        71.70 2255190716  894753892
dm-2             10.40        28.47        31.55  355349885  393682204
dm-3              8.94         2.31        34.80   28841385  434281596
{% endhighlight %}
tps         该设备每秒的传输次数  
kB_read/s   每秒从设备读取的数据量  
kB_wrtn/s   每秒向设备写入的数据量  
kB_read     读取的总数据量  
kB_wrtn     写入的总数量数据量  


{% highlight sh linenos %}
# iostat -xm
Linux 2.6.32-220.el6.x86_64 (belinkdev) 	2013年11月04日 	_x86_64_	(4 CPU)

avg-cpu:  %user   %nice %system %iowait  %steal   %idle
           2.21    0.00    0.49    1.89    0.00   95.41

Device:         rrqm/s   wrqm/s     r/s     w/s    rMB/s    wMB/s avgrq-sz avgqu-sz   await  svctm  %util
sda              36.51    27.46   13.11   13.21     0.23     0.16    30.33     0.00   13.12   2.63   6.92
dm-0              0.00     0.00    1.72    6.18     0.02     0.02    12.54     0.10   12.97   2.71   2.15
dm-1              0.00     0.00   45.18   17.92     0.18     0.07     8.00     0.08    6.76   0.72   4.56
dm-2              0.00     0.00    2.51    7.89     0.03     0.03    11.55     0.06    5.79   1.50   1.56
dm-3              0.00     0.00    0.24    8.70     0.00     0.03     8.30     0.27   30.23   1.90   1.70

{% endhighlight %}
rrqm/s      每秒这个设备相关的读取请求有多少被Merge了（当系统调用需要读取数据的时候，VFS将请求发到各个FS，如果FS发现不同的读取请求读取的是相同Block的数据，FS会将这个请求合并Merge）  
wrqm/s      每秒这个设备相关的写入请求有多少被Merge了
r/s         每秒完成的读设备次数
w/s         每秒完成的写设备次数
rMB/s       每秒读M字节数
wMB/s       每秒写M字节数
avgrq-sz    平均每次设备I/O操作的数据大小
avgqu-sz    平均I/O队列长度
await       平均每次设备I/O操作的等待时间 (毫秒)
svctm       平均每次设备I/O操作的服务时间 (毫秒)
%util       I/O的系统利用率(一秒中有百分之多少的时间用于 I/O 操作)
