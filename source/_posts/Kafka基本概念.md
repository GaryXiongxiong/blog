---
title: Kafka基本概念
date: 2020-09-02 10:33:37
tags:
- Kafka
categories:
- blog
---
## 定义

- 分布式
- 发布/订阅模式
- 消息队列
- 可持久化

## 基本概念

#### 代理 Broker：
 一个Kafka进程（实例）被称作一个代理节点。在分布式集群中，一个服务器上部署一个broker。

#### 生产者 Producer：
Producer将消息记录发送到Kafka集群指定的主题（Topic）中存储。Producer也可通过算法决定将小幅发送到哪个分区（Partition）。
#### 消费者 Consumer：
Consumer 从Kafka集群中指定的Topic读取消息。

#### 消费者组 Consumer Group：
一个消费者组可以包括一个或多个消费者。一般而言一个消费者对应一个线程。在设置线程时需遵循“线程数小于等于分区数”。若线程数大于分区数，则多出的线程不会消费数据，造成资源浪费。

#### 主题 Topic：
Kafka消息的逻辑区分方式，用以存储不同业务类型的消息记录。

#### 分区 Partition：
Kafka消息的物理分区，不同分区对应不同数据文件。每个主题可以含有多个分区。Kafka通过分区实现并发读写。分区中的消息记录是有序的，每个消息都会有一个偏移量序号（offset）。一个分区对应一个Broker，一个Broker负责管理多个分区。

#### 副本 Replication：
在Kafka中，每个Topic都会在创建时要求指定副本数，默认为1。副本保证了Kafka的高可用性。

#### 记录 Record：
被实际写入Kafka的可被消费者读取的数据被称为记录。每个记录包含key-value 与 timestamp。


> **副本系数：**
>
> 当集群数>=3, 副本系数可设为3
>
> 若集群数<3, 副本系数可设为集群数量

## 工作机制

核心机制：消费者-生产者

![kafka-process](/images/kafka-process.png)

