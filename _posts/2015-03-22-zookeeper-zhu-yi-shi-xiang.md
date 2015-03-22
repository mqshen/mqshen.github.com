---
layout: post
title: "Zookeeper注意事项"
description: "Zookeeper一些需要注意的地方"
category: ""
tags: []
---
{% include JB/setup %}

##Session Recovery    
当Session恢复时Zookeeper的状态可能已经改变，所以client不应该持久化Zookeeper的状态缓存。    
Client在崩溃时可能已经完成了操作，所以client需要做一些清理工作。    

##当Znode重新创建时它的版本会被重置    


##sync call
Client可能已经从其它渠道获取到Zookeeper的状态修改。但当前client可能还没有从Zookeeper上知道这个事件。我们就需要使用sync这个异步请求使得这个路径的Znode活的同步。     

##多线程和同步API的顺序问题    

##同步和异步请求间的顺序问题    
通常来说混合使用同步和异步请求是不好的主意。但有一些例外：在启动时你需要确定Zookeeper中已经有一些数据。    

##Data and Child Limits     
Zookeeper每个请求的数据大小限制在1MB左右，如果有特殊的需求可以在 Unsafe Options 中就该这个词修改这个值。出于性能考虑我们不应该接近这个值。    


