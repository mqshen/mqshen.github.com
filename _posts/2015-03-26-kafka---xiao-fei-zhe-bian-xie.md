---
layout: post
title: "Kafka  消费者编写"
description: ""
category: "Kafka"
tags: [kafka]
---
{% include JB/setup %}

##高层消费者API    
高层接口一般用在在不需要处理消息的偏移量的时候。     
高层消费者把针对不同分区的最后的偏移量存储在Zookeeper中。这个偏移量是针对消费者组的(组名是唯一的)。    

###下面是关于消费者的比较重要的几个class     
KafkaStream  由ConsumerConnector返回。可以用来创建流式messages iterator。    
ConsumerConfig  与Zookeeper建立连接所需要的一些属性。    
ConsumerConnector  与Zookeeper建立的连接


    val props = new Properties()
    props.put("group.id", groupId)
    props.put("zookeeper.connect", zookeeperConnect)
    props.put("auto.offset.reset", if(readFromStartOfStream) "smallest" else "largest")
    props.put("zookeeper.session.timeout.ms", "500")
    props.put("zookeeper.sync.time.ms", "250")
    props.put("auto.commit.interval.ms", "1000")
    
    val config = new ConsumerConfig(props)

auto.commit.interval.ms 向Zookeeper提交偏移量的时间间隔。    

    val filterSpec = new Whitelist(topic)

    info("setup:start topic=%s for zk=%s and groupId=%s".format(topic,zookeeperConnect,groupId))
    val stream = connector.createMessageStreamsByFilter(filterSpec, 1, new DefaultDecoder(), new DefaultDecoder()).get(0)
    info("setup:complete topic=%s for zk=%s and groupId=%s".format(topic,zookeeperConnect,groupId))

    def read(write: (Array[Byte])=>Unit) = {
      info("reading on stream now")
      for(messageAndTopic <- stream) {
        try {
          info("writing from stream")
          write(messageAndTopic.message)
          info("written to stream")
        } catch {
          case e: Throwable =>
            if (true) { //this is objective even how to conditionalize on it
              error("Error processing message, skipping this message: ", e)
            } else {
              throw e
            }
        }
      }
    }


