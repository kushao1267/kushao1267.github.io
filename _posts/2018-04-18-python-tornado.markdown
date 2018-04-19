---
layout:     post
title:      "由tornado引发的一些思考"
date:       2018-04-18 21:57:00
author:     "Jonliu"
header-img: "img/post-bg-2015.jpg"
tags:
    - Python
---

> “Yeah It's cool. ”


## 前言
最近接触到公司的长轮训项目，用于实现好友列表页消息实时更新，看到组里的小伙伴是用tornado写的项目，所以进行一些研究。

<p id = "build"></p>
---

## 关于epoll
epoll是Linux内核为处理大批量文件描述符而作了改进的poll，是Linux下多路复用IO接口select/poll的增强版本：
- 它能显著提高程序在`大量并发连接中只有少量活跃`的情况下的`系统CPU利用率`。

- 原因就是获取事件的时候，它无须遍历`整个被侦听的文件描述符集`，只要遍历那些被`内核IO事件`异步唤醒而加入`Ready队列`的描述符集合就行了。

tornado是通过epoll方式实现的异步IO框架，所以它虽然是单进程单线程，但是可以通过异步IO的方式保持成千上万的连接。

## web server与web app
WSGI的全称是Web Server Gateway Interface，这是一个规范，约定了web服务器`怎么调用`web应用程序的代码、web应用`如何处理`请求(比如常用的wsgi:gunicorn的-k网络模型，-w进程数，-b绑定的host:port，-c配置web app的文件等等)。

`只要web应用程序和web服务器都遵守WSGI 协议，那么，web应用程序和web服务器就可以随意的组合`。正是因为WSGI规范，所以无论是什么语言实现的什么web框架都可以与nginx拼接。

![image](/img/in-post/python-tornado-1.jpeg)

其中web server通常是nginx/apache,WSGI可以是gunicorn/uwsgi/tornado, web框架可以是django
由于tornado实现了web server和wsgi协议，所以可以通过如下方式提高服务器的效率：
```
1 * nginx -> n * wsgi(tornado实现) 

1 * wsgi-> n * django(web app)进程

说明：nginx是多进程的所以能起多个web app进行负载均衡,WSGI接口对应多个web app worker,gunicorn也支持epoll
```
另外：tornado在实现websocket和长轮训方面有着天然的优势(例如：社交产品中，server与client端保持长轮训，实时更新用户消息.游戏服务器中websocket保持实时状态)

## 关于websocket和长轮训
- 长轮训通常是发出http请求，等待回应，此时并不断开连接，所以io阻塞，而tornado实现了异步io，在长轮训保持不活跃的连接时，tornado可以异步地去建立其他的连接

- websocket客户端发出一个较大http请求，其中包含...,与服务器建立连接，一旦建立连接，中间的数据传输使用tcp协议通信数据，不过每个数据包会多几个bytes的websocket协议头。同理，websocket也是建立了一个长连接，所以异步io的tornado非常适合！
