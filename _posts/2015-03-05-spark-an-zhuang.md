---
layout: post
title: "Spark安装"
description: ""
category: "Spark"
tags: [spark, cluster]
---
{% include JB/setup %}
##Spark用HDFS做数据存储所以我们先安装hadoop
###hadoop安装
1.master到slave的免登入SSH配置。

2.修改master的 $hadoop_home/etc/hadoop/slaves 如：
               
    slave1
    slave2
    slave3


3.修改master和slaves的 $hadoop_home/etc/hadoop/core_site.xml 如：    

    <configuration>
        <property>
                <name>fs.defaultFS</name>
                <value>hdfs://master:9000</value>
        </property>
        <property>
                <name>io.file.buffer.size</name>
                <value>131072</value>
        </property>
        <property>
                <name>hadoop.tmp.dir</name>
                <value>file:/app/hdfs/tmp</value>
                <description>Abase for other temporary directories.</description>
        </property>
        <property>
               <name>hadoop.proxyuser.hduser.hosts</name>
               <value>*</value>
       </property>
       <property>
               <name>hadoop.proxyuser.hduser.groups</name>
               <value>*</value>
       </property>
    </configuration>

4.修改master和slaves的 $hadoop_home/etc/hadoop/hdfs_site.xml 如：
    
    <configuration>
        <property>
                <name>dfs.namenode.secondary.http-address</name>
                <value>master:9001</value>
        </property>
        <property>
                <name>dfs.namenode.name.dir</name>
                <value>file:/app/hdfs</value>
        </property>
        <property>
                <name>dfs.datanode.data.dir</name>
                <value>file:/app/hdfs/data</value>
        </property>
        <property>
                <name>dfs.replication</name>
                <value>3</value>
        </property>
        <property>
                <name>dfs.webhdfs.enabled</name>
                <value>true</value>
        </property>
    </configuration>

5.格式化namenode 
    
    bin/hadoop namenode -format

6.启动hdfs
    
    sbin/start-dfs.sh

查看master上是否有NameNode进程slave上是否有DataNode进程。

7.日志chakan
hadoop的日志在$hadoop_home/logs/目录下


