---
title: Kafka 消费者篇
date: 2020-09-03 19:49:10
tags:
- Kafka
categories:
- blog
---

## 消费者与消费者组

消费者组是若干个消费者组成的集合，一个消费者组包含以下特性：

- 一个消费者组可以有一个或多个消费者实例
- 消费者组名（GroupId）通常由一个一个字符串表示，有唯一性。
- 当一个消费者组订阅了主题，那么该主题中的每个分区职能分配给否一个消费者组中的某一个消费者程序

一个分区对应一个消费者，一个消费者可以负责多个分区。消费者数量尽量不要超过话题的分区数，否则多出的消费者将处于空闲状态。

![image-20200904103005821](/images/kafka-consumer.png)

## 使用脚本控制消费者

```bash
./kafka-console-consumer.sh --broker-list broker_name:port --topic topic_name
```

## 使用消费者API

```java
import org.apache.kafka.clients.producer.KafkaConsumer;
import org.apache.kafka.clients.producer.ConsumerRecords;

import java.util.Properties;

public class KafkaConsumerDemo {
    public static void main(String[] args) {
        //创建并写入客户端配置
        Properties props = new Properties();
        //Kafka 集群地址合集
        props.put("bootstrap.servers", "localhost:9092");
        //Kafka 消费者组id
        props.put("group.id", "CountryCounter");
        //反序列化器
        props.put("key.deserializer", "org.apache.kafka.common.serializaiton.StrignDeserializer");
        props.put("value.deserializer", "org.apache.kafka.common.serializaiton.StrignDeserializer");
		
        //根据配置创建consumer
       	KafkaConsumer<String, String> consumer = new KafkaConsumer<String, String>(props);
        //订阅topic
       	consumer.subscribe(Collections.singletonList("test"));
        try {
            while (true) {
                // 100 是超时时间（ms），在该时间内 poll 会等待服务器返回数据
                ConsumerReccords<String, String> records = consumer.poll(100); 
                
                // poll 返回一个记录列表。
                // 每条记录都包含了记录所属主题的信息、记录所在分区的信息、记录在分区里的偏移量，以及记录的键值对。
                for (ConsumerReccord<String, String> record : records) {
                    log.debug("topic=%s, partition=%s, offset=%d, key=%s, value=%s",
                        record.topic(), record.partition(), record.offset(), 
                        record.key(), record.value());
                }
            }
        } finally {
            // 关闭消费者,网络连接和 socket 也会随之关闭，并立即触发一次再均衡
            consumer.close();
        }	
    }
}
```

KafkaConsumer是非多线程并发安全的：如果多个线程公用一个KafkaConsumer实例，则抛出异常错误信息。

