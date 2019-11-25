---
layout:     post
title:      "布隆过滤器理解及实现"
date:       2019-11-25 19:07:43
author:     "Jonliu"
header-img: "img/post-bg-2018.jpg"
tags:
    - System
---

### 应用场景
- 1.去重
- 2.避免缓存穿透


### 实现原理
BloomFiler又叫布隆过滤器，下面举例说明BlooFilter实现原理：
> 比如你有10个Url，你完全可以创建一长度是100bit的数组，然后对url分别用5个不同的hash函数进行hash，得到5个hash后的值，这5个值尽可能的保证均匀分布在100个bit的范围内。

> 然后把5个hash值对应的bit位都置为1，判断一个url是否已经存在时，一次看5个bit位是否为1就可以了，如果有任何一个不为1，那么说明这个url不存在。这里需要注意的是，如果对应的bit位值都为1，那么也不能肯定这个url一定存在，这个是BloomFilter的特点之一。

![举例](http://oss.lanjingdejia.com/file/2018/7/d2fb6f30cbe3430db4eeb6bb7752c0be-image.png)


### 核心思想

- 多个hash，增大随机性，减少hash碰撞的概率

- 扩大数组范围，使hash值均匀分布，进一步减少hash碰撞的概率。


### Python实现代码
[BloomFilterRedis](https://github.com/kongtianyi/BloomFilterRedis/tree/master/BloomFilterRedis)