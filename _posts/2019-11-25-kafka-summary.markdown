---
layout:     post
title:      "Kafka要点总结"
date:       2019-11-25 19:00:43
author:     "Jonliu"
header-img: "img/post-bg-2018.jpg"
tags:
    - System
---

### Kafka介绍
优点:
- 高吞吐: 单机10w+/s，高于其他消息中间件。
- 高容错：多副本(Replica)，允许单机宕机而不丢失数据。
- 高性能：顺序写、零拷贝、分片等特性。
- 高并发: Broker使用优化的reactor网络线程模型。

应用场景:
- 需要异步、削峰、解耦等场景。
- 大数据、实时计算、日志分析等高吞吐量要求的场景。(但RabbitMQ消息延时更低)

### 系统结构
- Controller:
在分布式系统中大多数都是主从式的架构，个别是对等式的架构，比如ElasticSearch。kafka也是主从式的架构，主节点是controller，其余的为从节点，controller是需要和zookeeper进行配合管理整个kafka集群。

- Topic:
topic代表一种业务维度的消息，可分布在多个partition中。

- Partition:
一个topic的消息分布在多个partion中，而partition又在多个host中，这样就增加了多线程消费能力，大大提高了数据处理的吞吐量及性能。

- Broker:
所有储存队列消息的主机，共同组成Broker，是生产数据与消费数据的中间部分。

- Host:
Host也称为Broker节点，Kafka对主机硬件配置要求不高，非常容易扩容，每个主机可有多个partition，可认为partition是主机上的文件夹。

- Replica:
kafka在0.8版本后加入了副本机制，为了保证数据安全，每个partition可以设置多个数据副本，一般设置两个副本即一个leader一个fllower。

- Consumer Group:
通过GroupID来区分不同的Consumer Group，多个进程设置为一个GroupID可以实现并发消费数据，提高性能，同一个Group内的数据不会被重复消费；多个进程设置为不同GroupID可以实现一份数据重复消费，Group之间互不干扰。

![消息结构图](http://ww1.sinaimg.cn/large/961a76f7ly1g9aezxsjpaj20lx0g976j.jpg)

### 选举机制
- 原理：
Raft选举机制。

- 工作机制：
每个副本都有角色之分，它们会选举一个副本作为leader，而其余的作为follower，我们的生产者在发送数据的时候，是直接发送到leader partition里面，然后follower partition会去leader那里自行同步数据，消费者消费数据的时候，也是从`leader`那去消费数据的。


### 参考
- 《Kafka权威指南》
- [浅谈Kafka选举机制](https://blog.csdn.net/qq_37142346/article/details/91349100)
- [插曲：大白话带你认识Kafka](https://juejin.im/post/5dcf6b6e51882510a23314f3)