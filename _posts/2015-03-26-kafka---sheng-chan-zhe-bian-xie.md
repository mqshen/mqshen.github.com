---
layout: post
title: "Kafka  生产者编写"
description: ""
category: "Kafka"
tags: [kafka]
---
{% include JB/setup %}

##一个简单的生产者    

###属性定义    

    val props = new Properties()
    
    val codec = if(compress) DefaultCompressionCodec.codec else NoCompressionCodec.codec
    
    props.put("compression.codec", codec.toString)
    props.put("serializer.class", codec.toString)
    props.put("producer.type", if(synchronously) "sync" else "async")
    props.put("metadata.broker.list", brokerList)
    props.put("batch.num.messages", batchSize.toString)
    props.put("message.send.max.retries", messageSendMaxRetries.toString)
    props.put("request.required.acks",requestRequiredAcks.toString)
    props.put("client.id",clientId.toString)

metadata.broker.list 定义的生产者所要连接的的broker地址<node:port>       

serializer.class 定义发送到broker的消息序列化所采用的class。默认情况下key和message所采用的类是一样的，但也可以指定key. serializer.class属性。     

request.required.acks 当broker收到消息时是否需要发送ack。(0:不需要ack,1:当broker收到消息时发送ack,-1:当所有在in-sync状态的replicas收到数据后才发送ack。    

partitioner.class 定义message分区的class。如果key为null，Kafka将对消息采用随机分区方法。    

producer.type 消息发送的模式，sync还是async。    




###发送消息    

    val producer = new Producer[String, String](new ProducerConfig(props))
    producer.send(kafkaMessage(message, partition))

分区类

    class MyPartitioner extends Partitioner {
    
      override def partition(key: Any, numPartitions: Int): Int = {
        key match {
          case e: Int if e > 0 =>
            e % numPartitions
          case _ =>
            0
        }
      }
    
    }
