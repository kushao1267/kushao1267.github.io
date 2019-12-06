---
layout:     post
title:      "RabbitMQ踩坑总结"
date:       2019-07-01 14:11:42
author:     "Jonliu"
header-img: "img/post-bg-2018.jpg"
tags:
    - System
---

## RabbitMQ的持久化

- 1.设置Exchange

- 2.MessageQueue的durable属性为true，可以使队列和Exchange持久化

- 3.生产者在发送消息的时候，将delivery_mode设置为2（persistent）

## RabbitMQ的事务机制

在许多场景下，你不能丢失消息，包括producer发布到RabbitMQ这个链路。常用的方式是打开RabbitMQ的事务机制，事务需要保证数据的一致性，因此将经过:

- 1.broker持久化消息
- 2.consumer收到消息
- 3.publisher知道消息已经成功consume

每个消息都必须经历以上三个步骤，才算一次事务成功，它是同步的。如果采用事务，发布消息的性能将变得很差，官方甚至描述说：发布1w条消息花费了4分多钟，可想而知有多慢。

因此，我们可以采用异步的方式来解决此问题，官方提供了一个轻量级的AMQP扩展: Lightweight Publisher Confirms.[文档](https://www.rabbitmq.com/blog/2011/02/10/introducing-publisher-confirms/)

publisher发送消息后不等待，而是异步监听是否成功。这种方式又分为两种模式，一种是return，另一种是confirm. 前一种是publisher -> exchange后，异步收到消息。第二种是publisher发送消息到exchange -> queue -> consumer收到消息后才会收到异步收到消息。可见，第二种方式更加安全可靠。

![图1](http://ww3.sinaimg.cn/bmiddle/60c9620fjw1epmt7nicy0j20fp08kjs2.jpg)

异步方式的效率是事务方式效率的近100倍。

> It takes a bit more than 2 seconds to publish 10000 messages.

但是，异步也存在些局限性。如果一旦出现broker挂机或者网络不稳定，broker已经成功接收消息，但是publisher并没有收到confirm或return。这时，对于publisher来说，只能重发消息解决问题。

## RabbitMQ的消息确认机制

- 当消费者发回ack（nowledgement）告诉RabbitMQ已接受并处理了该消息，RabbitMQ则删除它。

- 当消费者发回reject（Negative nowledgement）告诉RabbitMQ，RabbitMQ认为worker拒绝了消息，此时RabbitMQ会requeue该消息或者直接删除，但每次只能一条reject。

- 当消费者发回nack（Negative nowledgement）与reject表现一致，但是支持批量。

> 如果消费者在收到消息后断开了连接还未ack，消息服务器会重传该消息给下一个订阅者(重发)，如果没有订阅者就会存储该消息。因此，可能会遇到重复处理消息的问题。

> 在RabbitMQ中，没有消息延迟(timeout)的概念，task在某个worker的连接断掉之后，才会重发。即使一个task执行了很长很长的时候，也是正常的。RabbitMQ会一直等到处理某个消息的“Consumer”的链接失去之后，才确定这个消息没有正确处理，从而RabbitMQ重发这个消息。消息确认机制(Message acknowledgment)是默认关闭的。如果就我们的应用需要确保消息可靠性，即在消息的任务处理完之后再ack，记得在Consumer打开它的开关。

## headers工作模式

header-exchange(头交换机)和topic-exchange(主题交换机)有点相似，但与topic-exchange不同的是路由是基于routing key，而header-exchange的路由值基于消息的header数据。
消息根据publish时所带properties中的headers参数进行分发路由，其中headers字典可自定义业务键值。

> 可利用发布消息时在headers中传递一些业务类型的键值，例如：重试次数(x-retry-times)，任务开始时间(x-start-time)，到达我们重试次数和时间控制的目的。此外，使用OpenTracing的小伙伴还可以在headers中添加tracer数据，便于链路追踪。

重试次数控制的代码示例:

1.发布消息

```python

channel.basic_publish(exchange=exchange,
                      routing_key=routing_key,
                      properties=pika.BasicProperties(
                          headers={'x-retry-times': 3}
                      ),
                      body=message)
```

2.监听消息

```python
def on_message(channel: pika.adapters.blocking_connection.BlockingChannel,
               method_frame: pika.spec.Basic.Deliver,
               header_frame: pika.spec.BasicProperties,
               body: typing.Union[str, bytes]):

            retries = header_frame.headers['x-retry-times']
            # if excceed retry times.
            if retries >= MAX_RETRIES:
                channel.basic_ack(delivery_tag=method_frame.delivery_tag)
            # TODO: else do work
            channel.basic_nack(delivery_tag=method_frame.delivery_tag)
```

## 使用python编写RabbitMQ的代码

建议参考:

- [异步消费者示例](https://github.com/pika/pika/blob/master/examples/asynchronous_consumer_example.py)

- [异步发布者示例](https://github.com/pika/pika/blob/master/examples/asynchronous_publisher_example.py)

## 参考

- [1][RabbitMQ的消息确认参数](https://www.rabbitmq.com/nack.html)

- [2][官方代码示例](https://github.com/pika/pika/blob/master/examples)
