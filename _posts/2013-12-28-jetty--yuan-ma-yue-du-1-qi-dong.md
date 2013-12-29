---
layout: post
title: "Jetty 源码阅读(1)启动 完善中..."
description: "主要介绍下环境准备、启动加载的配置文件的涵义"
category: ""
tags: [Jetty]
---
{% include JB/setup %}

主要介绍下环境准备、启动加载的配置文件的涵义.

从[eclipse](https://github.com/eclipse/jetty.project.git)git仓库中下载源码。导入eclipse。    

org.eclipse.jetty.start.Main为Jetty的启动类。  

为了调试方便设置了它的启动JVM参数为：
    
    -Djetty.home= /jetty-start/src/test/resources/usecases/home

启动参数为:

    --module=http,annotations,deploy --debug

并把test-integration工程下得配置文件和demo拷贝到jetty.home下

由于jetty-http.xml的port设置的是0这样每次启动的端口都不一样。修改成

    <Set name="port">8080</Set>
    
启动顺序如下图 其中的异步调用没有标识出来'

![jetty start]({{ BASE_PATH }}/assets/images/JettyStart.png)

jetty接受http请求顺序

![jetty http]({{ BASE_PATH }}/assets/images/JettyHttp.png)
