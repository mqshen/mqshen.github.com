---
layout: post
title: "Zookeeper 处理失败"
description: "zookeeper的失败处理机制"
category: "Zookeeper"
tags: [zookeeper, distribute]
---
{% include JB/setup %}

##介绍    
错误出现的三个主要地方：    
1. Zookeeper服务器     
2. 网络    
3. 应用程序    

Zookeeper提供两种不同的失败类型：     
1. 可恢复的     
2. 不可恢复的(session超时，授权收回等)    

当客户端重新连接到Zookeeper上时会比对znode的时间戳如果监听的时间戳比客户端的晚就会触发监听事件。
这个逻辑在除了exists的其它方面工作的很好。    
exists:如下图C<sub>2</sub>就不能监听到C<sub>1</sub>对znode event的操作

![failure]({{ BASE_PATH }}/assets/images/zookeeper/failure.png)

##选举与外部资源    

###fencing(用来解决外部资源的冲突)    

![fencing]({{ BASE_PATH }}/assets/images/zookeeper/fencing.png)     

要支持这种机制就需要修改客户端与资源之间的协议，外部资源需要存储和跟踪最后的zxid。
