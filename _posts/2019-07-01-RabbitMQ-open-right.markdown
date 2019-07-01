---
layout:     post
title:      "RabbitMQ踩坑总结"
date:       2019-06-29 14:11:42
author:     "Jonliu"
header-img: "img/post-bg-2018.jpg"
tags:
    - RabbitMQ
    - Python
---

## RabbitMQ的消息确认

- 当消费者发回ack（nowledgement）告诉RabbitMQ已接受并处理了该消息，RabbitMQ则删除它。

- 当消费者发回reject（Negative nowledgement）告诉RabbitMQ，RabbitMQ认为worker拒绝了消息，此时RabbitMQ会requeue该消息或者直接删除，但每次只能一条reject。

- 当消费者发回nack（Negative nowledgement）与reject表现一致，但是支持批量。

## headers工作模式

header-exchange(头交换机)和topic-exchange(主题交换机)有点相似，但与topic-exchange不同的是路由是基于routing key，而header-exchange的路由值基于消息的header数据。
消息根据publish时所带properties中的headers参数进行分发路由，其中headers字典可自定义业务键值。

> 可利用发布消息时在headers中传递一些业务类型的键值，例如：重试次数(x-retry-times)，任务开始时间(x-start-time)，到达我们重试次数和时间控制的目的。
> 此外，使用OpenTracing的小伙伴还可以在headers中添加tracer数据，便于链路追踪。

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
            channel.basic_ack(delivery_tag=method_frame.delivery_tag)
```

## 使用python编写RabbitMQ的代码

建议参考:

- [异步消费者示例](https://github.com/pika/pika/blob/master/examples/asynchronous_consumer_example.py)
- [异步发布者示例](https://github.com/pika/pika/blob/master/examples/asynchronous_publisher_example.py)

## 参考

[1][RabbitMQ的消息确认参数](https://www.rabbitmq.com/nack.html)
[2][官方代码示例](https://github.com/pika/pika/blob/master/examples)
