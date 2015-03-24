---
layout: post
title: "kafka简介"
description: ""
category: "kafka"
tags: [kafka]
---
{% include JB/setup %}

下图是典型的kafka支撑的大数据收集分析场景。    
![introduce]({{ BASE_PATH }}/assets/images/kafka/introduce.png)     

一般生产者有下面几种：    
1. 前端web应用产生的应用日志    
2. 代理产生的web分析日志    
3. 适配器产生的变换日志    
4. 服务所产生的调用跟踪日志    

消费者分为：    
1. 离线消费者（Spark，Hadoop，传统数据仓库的离线分析）
2. 近实时消费者（接收的消息存放在HBase或Cassandra这样的NoSQL数据库中来进行近实时的分析）    
3. 实时分析（在内存数据库中消费，过滤消息并向相关的实体发送事件）    


