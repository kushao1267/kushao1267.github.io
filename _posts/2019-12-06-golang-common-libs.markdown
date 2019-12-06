---
layout:     post
title:      "golang常用库及组件"
date:       2019-12-06 10:00:43
author:     "Jonliu"
header-img: "img/post-bg-2018.jpg"
tags:
    - System
---

Log: logrus, zap(比logrus性能强, uber-go维护)

Redis: https://github.com/go-redis/redis (支持cluster)

Config: viper, github.com/BurntSushi/toml

Yaml:  [yaml](github.com/ghodss/yaml)

Testing: goconvey

Orm: gorm

Web framework: gin/beego/echo

Request: grequests

Server register/discovery: Consul

microservice: go-micro, grpc

Compile: bazel

CLI: [cobra](https://github.com/spf13/cobra)

服务监控(prometheus): https://github.com/prometheus/client_golang, ginprom(gin的promethus插件)

Kafka client: https://github.com/Shopify/sarama

DI(依赖注入): wire(google), dig(uber)