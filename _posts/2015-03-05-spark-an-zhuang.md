---
layout: post
title: "Spark安装"
description: "初次安装Spark记录"
category: "Spark"
tags: [spark, cluster]
---
{% include JB/setup %}
系统为CentOS

#####Spark用HDFS做数据存储所以我们先安装hadoop#####    

###hadoop安装###    

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

打开 http://master:50070 可以看到hadoop的状态
查看master上是否有NameNode进程slave上是否有DataNode进程。

7.日志chakan

hadoop的日志在$hadoop_home/logs/目录下


#####Spark Cluster使用Mesos作为Cluster Manger所以我们接下来安装Mesos#####

###Mesos安装###
为系统添加YUM源    
新建文件 /etc/yum.repos.d/wandisco-svn.repo 内容为:
    
    [WandiscoSVN]
    name=Wandisco SVN Repo
    baseurl=http://opensource.wandisco.com/centos/6/svn-1.8/RPMS/$basearch/
    enabled=1
    gpgcheck=0

执行
    
    $ sudo yum groupinstall -y "Development Tools"
    $ sudo yum install -y python-devel java-1.7.0-openjdk-devel zlib-devel libcurl-devel openssl-devel cyrus-sasl-devel cyrus-sasl-md5 apr-devel subversion-devel apr-utils-devel

Mesos编译需要maven所以我们还需要安装maven这里就不多说了

新版本的Mesos需要gcc 4.8以上这需要添加新的依赖

    wget http://people.centos.org/tru/devtools-2/devtools-2.repo -O /etc/yum.repos.d/devtools-2.repo
    yum install devtoolset-2-gcc devtoolset-2-binutils devtoolset-2-gcc-c++

然后使用

    scl enable devtoolset-2 bash 

来启用devtoolset-2

再用make工具进行安装
    
    $ cd mesos
    $ ./bootstrap  #只用在从git仓库下载的源码才需要执行这一行
    $ mkdir build
    $ cd build
    $ ../configure
    $ make
    $ make check #进行测试
    $ make install

启动Mesos
master
    
    mesos-master --work_dir=/var/lib/mesos -ip=master_ip

slaves

    mesos-slave --master=master_ip:5050 --ip=slave_ip

这是我们打开  http://master:5050 就可以看到Mesos Cluster的状态


##Spark 安装
由于我的HADOOP是2.6.0版本这就需要我们在构建Spark时需要

1.指定hadoop.version=2.6.0不然就会出现

    Server IPC version 9 cannot communicate with client version 4

2.修改protobuf.version=2.5.0不然就会出现

    class org.apache.hadoop.hdfs.protocol.proto.ClientNamenodeProtocolProtos$AddBlockRequestProto overrides final method getUnknownFields.()Lcom/google/protobuf/UnknownFieldSet

把构建好的版本放到hdfs上

    $ bin/hadoop fs -put ~/server/spark-1.2.1-bin-1.0.4.tgz /spark-1.2.1-bin-1.0.4.tgz

修改spark-env.sh:
    
    export MESOS_NATIVE_LIBRARY=/usr/local/lib/libmesos.so
    export SPARK_EXECUTOR_URI=hdfs://${hdfs name node host}:9000/spark-1.2.1-bin-1.0.4.tgz

测试spark shell:
    
    bin/spark-shell --master mesos://host:5050

    

