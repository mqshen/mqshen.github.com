---
layout: post
title: "Jetty源码阅读 体系架构"
description: "Jetty是由一个或多个connector核心组件以及一些列handler组件和一个线程池组成."
category: ""
tags: [Jetty]
---
{% include JB/setup %}

###基本结构
Jetty是由一个或多个connector核心组件以及一些列handler组件和一个线程池组成.
Connector负责监听接收客户连接请求而handler组件则负责处理请求并给予响应，这两个组件工作所需要的线程资源都直接从线程池ThreadPool中获取。    
如下图:
![Jetty Architecture]({{ BASE_PATH }}/assets/images/JettyArchitecture.png)

###组件生命周期
对于Jetty来说，每个组件都有其生命周期，Jetty采用了统一的LifeCyle接口来控制，
![Jetty Life Cycle]({{ BASE_PATH }}/assets/images/JettyLifeCycle.png)

connector,handler等组件全部都直接或间接实现了LifeCyle接口.它们是嵌套链式结构的。

