---
layout: post
title: "源码阅读 （1）概览"
description: ""
category: "spark"
tags: [spark]
---
{% include JB/setup %}

####基本概念

|| RDD || resilient distributed dataset ||
|| Operation || 作用在RDD上的transformation和action ||
|| Job || 包含多个RDD及作用在相应RDD上的Operation ||
|| Stage || 一个Job分为多个阶段 ||
|| Partition || 数据分区，一个RDD的数据可以分为多个不用的分区 ||
|| DAG || 有向无环图，反应RDD之间的依赖 ||
|| Narrow Dependency || 窄依赖，一个父RDD分区最多被一个子RDD分区引用，表现为一个父RDD的分区对应于一个子RDD的分区或多个父RDD的分区对应于一个子RDD的分区，也就是说一个父RDD的一个分区不可能对应一个子RDD的多个分区，如map、filter、union等操作则产生窄依赖 ||
|| Wide Dependency || 宽依赖，一个子RDD的分区依赖于父RDD的多个分区或所有分区，也就是说存在一个父RDD的一个分区对应一个子RDD的多个分区，如groupByKey等操作则产生宽依赖操作； ||
|| Cache Management || 缓存管理 ||

###Spark运行设计的组件

![组件]({{ BASE_PATH }}/assets/images/spark/overview.png)

###运行
![运行]({{ BASE_PATH }}/assets/images/spark/run.png)
