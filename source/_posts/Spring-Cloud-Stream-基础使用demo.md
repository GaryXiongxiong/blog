---
title: Spring Cloud Stream 配合Kafka 基础使用demo
date: 2020-09-08 11:08:39
tags:
- Spring Cloud
- Kafka
categories: 
- blog
---

## 简介与概念

Spring Cloud Stream 是一个用于构建消息驱动微服务的框架。Spring Cloud Stream 基于 Spring Boot，整合消息中间件（Kafka或RabbitMQ） 构建可独立运行，生产级的Spring应用。

#### 应用模型

一个**Spring Cloud Stream**应用程序依赖于于独立的消息中间件。应用通过**Spring Cloud Stream**注入的输入和输出*通道*与外部世界通信。通道通过专用的*Binder*实现与外部代理连接。这种模型屏蔽了消息中间件的使用差异，我们只需掌握Spring Cloud Stream的使用就可以方便的构建消息驱动的微服务应用。

![Spring Cloud Stream引用模型](/images/spring-cloud-stream-model.png)

#### Binder

Binder 是 Spring Cloud Stream 的一个抽象概念，是应用与消息中间件之间的粘合剂。目前 Spring Cloud Stream 实现了 Kafka 和 Rabbit MQ 的binder。

通过 binder ，可以很方便的连接中间件，可以动态的改变消息的
 destinations（对应于 Kafka 的topic，Rabbit MQ 的 exchanges），这些都可以通过`spring.cloud.stream.binder`进行配置。

#### 发布订阅模型

Spring Cloud Stream 使用了经典的发布/订阅模式。发布者将消息发布到指定的Topic中，订阅者通过订阅该Topic来消费消息。

#### Consumer Group

类似于Kafka中消费者组的概念，每个binding可以指定一个group，一条消息只会被同一个group中的一个binding消费。可以被用来防止重复消费。可以于`spring.cloud.stream.bindings.<channelName>.group`中定义。

## 搭建Kafka与Zookeeper

为了在本地构建我们的第一个Spring Cloud Stream应用，我们需要先行搭建其依赖的Kafka。Kafka又需要用到Zookeeper。这里使用Docker Compose来快速搭建Kafka与Zookeeper。

#### 编写 `Docker-compose.yml`

```yaml
version: '3'
services:
  kafka:
    image: wurstmeister/kafka
    container_name: kafka-streamlistener
    ports:
      - "9092:9092"
    environment:
      - KAFKA_ADVERTISED_HOST_NAME=192.168.99.100 #这里写docker宿主机地址。此例为在windows Docker Quickstart中运行，故地址为docker所在虚拟机的地址。
      - KAFKA_ADVERTISED_PORT=9092
      - KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181
    depends_on:
      - zookeeper
  zookeeper:
    image: wurstmeister/zookeeper
    ports:
      - "2181:2181"
    environment:
      - KAFKA_ADVERTISED_HOST_NAME=zookeeper
  kafka-manager:
    image: sheepkiller/kafka-manager
    ports:
      - 9000:9000
    environment:
      ZK_HOSTS: zookeeper:2181
      KAFKA_BROKERS: kafka:9092

```

#### 启动配置好的Docker-compose

```bash
docker-compose up -d
```

之后前往 `http://192.168.99.100:9000`验证 Kafka 运行情况

## 构建 Spring Cloud Stream 应用

本例中，我们将构建两个微服务，`Producer`开放Restful接口，并将接口请求发布至消息队列。`Sample`订阅队列并打印其中的消息。

#### 引入依赖

首先在两个项目中引入依赖

```xml
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-starter-stream-kafka</artifactId>
        </dependency>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
            <exclusions>
                <exclusion>
                    <groupId>org.junit.vintage</groupId>
                    <artifactId>junit-vintage-engine</artifactId>
                </exclusion>
            </exclusions>
        </dependency>
        <dependency>
            <groupId>org.springframework.cloud</groupId>
            <artifactId>spring-cloud-stream-test-support</artifactId>
            <scope>test</scope>
        </dependency>
```

#### 在 Spring Boot 中配置 Spring Cloud Stream 的 binder 与 bindings

Consumer的`application.yml`

```yaml
spring:
  cloud:
    stream:
      kafka:
        binder:
          # Kafka节点地址列表
          brokers: 192.168.99.100:9092
      default-binder: kafka
      bindings:
        # 设置binding
        msg_output:
          # 对应kafka的topic名称
          destination: msg
          binder: kafka
        # 可以设置多个binding，对应不同或相同的destination，如果destination相同且未设置group，将会重复消费对应destination中的消息。若group相同将会采用竞争策略，只有一个binding可以消费消息
        error_output:
          destination: error
          binder: kafka
```

Consumer的`application.yml`

```yaml
server:
  port: 8080
spring:
  cloud:
    stream:
      kafka:
        binder:
          brokers: 192.168.99.100:9092
      default-binder: kafka
      bindings:
        msg_input:
          destination: msg
          binder: kafka
        error_input:
          destination: error
          binder: kafka
```

#### `Producer`编写

定义binding接口

```java
public interface Producer {
    //定义通道名称
    String OUTPUT_CHANNEL ="msg_output";
    String ERROR_CHANNEL ="error_output";

    //通过Output注解定义输出通道
    @Output(Producer.OUTPUT_CHANNEL)
    MessageChannel messageOutput();

    @Output(Producer.ERROR_CHANNEL)
    MessageChannel errorOutput();
}
```

开启`binding`注解

```java
@SpringBootApplication
@EnableBinding(Producer.class)
public class StreamproducerApplication {
    public static void main(String[] args) {
        SpringApplication.run(StreamproducerApplication.class, args);
    }
}
```

定义Controller接收请求并置入消息队列

```java
@RestController
@RequestMapping("/")
public class Controller {

    @Resource
    Producer producer;

    @RequestMapping("/msg/{key}")
    public String sendMsg(@PathVariable String key, @RequestBody Object msg){
        Map<String,Object> payload = new HashMap<>();
        payload.put(key,msg);
        //将负载送入通道
        producer.messageOutput().send(MessageBuilder.withPayload(payload).build());
        return "sent";
    }

    @RequestMapping("/error/{key}")
    public String sendErr(@PathVariable String key, @RequestBody Object err){
        Map<String,Object> payload = new HashMap<>();
        payload.put(key,err);
        producer.errorOutput().send(MessageBuilder.withPayload(payload).build());
        return "sent";
    }
}
```

#### `Consumer`编写

定义binding接口

```java
public interface Consumer {
    String INPUT_CHANNEL ="msg_input";
    String ERROR_CHANNEL ="error_input";

    @Input(Consumer.INPUT_CHANNEL)
    SubscribableChannel messageInput();

    @Input(Consumer.ERROR_CHANNEL)
    SubscribableChannel errorInput();
}
```

使用`@StreamListener`注解消费消息

```java
@EnableBinding(Consumer.class)
public class Receiver {
    @StreamListener(Consumer.INPUT_CHANNEL)
    public void getInput(Map<String,Object> msg){
        System.out.println("[msg]: "+msg.toString());
    }
    @StreamListener(Consumer.ERROR_CHANNEL)
    public void getError(Map<String,Object> error){
        System.out.println("[err]: "+error.toString());
    }
}
```

#### 结果

Producer请求

```json
{
    "msg":"hello",
    "uid":"3319"
}
{
    "error":"ERROR",
    "uid":"2113"
}
```

Consumer输出

```
[msg]: {msg1={msg=hello, uid=3319}}
[err]: {error1={error=ERROR, uid=2113}}
```

## References

> - [Spring Cloud Stream中文指导手册](https://blog.csdn.net/qq_32734365/article/details/81413218)
> - [Spring Cloud Stream - Quick Start](https://cloud.spring.io/spring-cloud-static/spring-cloud-stream/3.0.6.RELEASE/reference/html/spring-cloud-stream.html#_quick_start)
> - [使用 Spring Cloud Stream 构建消息驱动微服务](https://www.jianshu.com/p/fb7d11c7f798)
