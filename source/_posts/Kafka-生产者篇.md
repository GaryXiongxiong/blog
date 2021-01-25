---
title: Kafka 生产者篇
date: 2020-09-02 15:06:35
tags:
- Kafka
categories:
- blog
---

生产者在业务中实时读取原始数据进行业务逻辑处理，然后调用Kafka的生产者接口将处理后的消息记录写入到Kafa集群中。

![kafka-producer-process](/images/kafka-producer-process.png)

## 使用脚本操作生产者

#### 生产者发布消息

```bash
./kafka-console-producer.sh --broker-list broker_name:port --topic topic_name
```

#### 消费者查看消息

```bash
./kafka-console-consumer.sh --broker-list broker_name:port --topic topic_name
```

## 使用Java API操作生产者

#### 导入依赖

```xml
		<dependency>
			<groupId>org.apache.kafka</groupId>
			<artifactId>kafka-clients</artifactId>
			<version>2.6.0</version>
		</dependency>
```

#### 使用生产者API

```java
import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.Producer;
import org.apache.kafka.clients.producer.ProducerRecord;

import java.util.Properties;

public class KafkaProducerDemo {
    public static void main(String[] args) {
        //创建并写入客户端配置
        Properties props = new Properties();
        //Kafka 集群地址合集
        props.put("bootstrap.servers", "localhost:9092");
        //对kafka节点应答的要求，0为不要求应答，1为需要一个节点应答，all为需要全部节点应答
        props.put("acks", "all");
        //发送失败重试次数
        props.put("retries", 0);
        //批处理量，减少请求次数
        props.put("batch.size", 16384);
        //增加延时，减少请求次数
        props.put("linger.ms", 1);
        //Producer可缓存数据的大小
        props.put("buffer.memory", 33554432);
        //key与value的序列化方式，用于实现Serializer接口
        props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
        props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");
		
        //根据配置创建producer
        Producer<String, String> producer = new KafkaProducer<String, String>(props);
        for (int i = 0; i < 100; i++) {
            //创建并发送record
            producer.send(new ProducerRecord<String, String>("my-topic", Integer.toString(i), Integer.toString(i))); //topic, key, value
        }

        producer.close();//关闭producer
    }
}
```

#### 多线程调用生产者API

由于Kafka的生产者对象是线程安全的，可以由多个线程调用Kafka生产者对象。

创建生产线程

```java
package kafka.producer;

import org.apache.kafka.clients.producer.KafkaProducer;
import org.apache.kafka.clients.producer.ProducerRecord;

/**
 * 生产者线程
 */
public class ProducerThread implements Runnable {

    private KafkaProducer<String, String> producer = null;
    private ProducerRecord<String, String> record = null;

    public ProducerThread(KafkaProducer<String, String> producer, ProducerRecord<String, String> record) {
        this.producer = producer;
        this.record = record;
    }

    @Override
    public void run() {
        producer.send(record, (metadata, e) -> {
            if (null != e) {
                e.printStackTrace();
            }
            if (null != metadata) {
                System.out.println("消息发送成功 ："+String.format("offset: %s, partition:%s, topic:%s  timestamp:%s", metadata.offset(), metadata.partition(), metadata.topic(), metadata.timestamp()));
            }
        });
    }

}
```

多个线程通过同一个Producer对象发送消息

```java
package kafka.producer;

import kafka.OrderMessage;
import kafka.partition.PartitionUtil;
import org.apache.kafka.clients.producer.*;
import org.apache.kafka.clients.producer.ProducerConfig;

import java.util.Properties;
import java.util.UUID;
import java.util.concurrent.*;

/**
 * 线程池生产者
 *
 * @author tangj
 * @date 2018/7/29 20:15
 */
public class ProducerDemo {
    static Properties properties = new Properties();

    static String topic = "test";

    static KafkaProducer<String, String> producer = null;

    // 核心池大小
    static int corePoolSize = 5;

    // 最大值
    static int maximumPoolSize = 20;

    // 无任务时存活时间
    static long keepAliveTime = 60;

    // 时间单位
    static TimeUnit timeUnit = TimeUnit.SECONDS;

    // 阻塞队列
    static BlockingQueue blockingQueue = new LinkedBlockingQueue();

    // 线程池
    static ExecutorService service = null;

    static {
        // 配置项
        properties.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        properties.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, PartitionUtil.class.getName());
        properties.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
        properties.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, "org.apache.kafka.common.serialization.StringSerializer");
        producer = new KafkaProducer<>(properties);
        // 初始化线程池
        service = new ThreadPoolExecutor(corePoolSize, maximumPoolSize, keepAliveTime, timeUnit, blockingQueue);
    }

    public static void main(String args[]) throws Exception {
        for (int i = 0; i < 6; i++) {
            service.submit(createMsgTask());
        }
    }


    /**
     * 生产消息
     *
     * @return
     */
    public static ProducerThread createMsgTask() {
        OrderMessage orderMessage = new OrderMessage();
        orderMessage.setId(UUID.randomUUID().toString());
        long timestamp = System.nanoTime();
        orderMessage.setCreateTime(timestamp);
        orderMessage.setRemake("rem");
        orderMessage.setsName("test");
        ProducerRecord<String, String> record = new ProducerRecord<String, String>(topic, timestamp + "", orderMessage.toString());
        ProducerThread task = new ProducerThread(producer, record);
        return task;
    }
}
```

## 序列化器

Kafka提供了一些序列化器，可以在`org.apache.kafka.common.serialization`中找到。

我们也可通过实现`org.apache.kafka.common.serialization.Serializer<T>`接口，并重写其中的`byte[] serialize(String topic, T data)`方法实现自定义序列化器。

## References
>
> - [Kafka并不难学](https://weread.qq.com/web/reader/bb03287071848770bb0d2c4kc81322c012c81e728d9d180)
> - [玩转Kafka的生产者——分区器与多线程](https://www.cnblogs.com/superfj/p/9440835.html)
> - [Kafka : Kafka入门教程和JAVA客户端使用](https://blog.csdn.net/sanyaoxu_2/article/details/80754134)