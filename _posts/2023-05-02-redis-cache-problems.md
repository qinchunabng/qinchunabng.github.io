---
layout: post
title: 缓存中常见的问题及的解决方案
categories: [Redis]
description: 缓存穿透的解决方法
keywords: Redis, 缓存
---

## 缓存穿透

缓存穿透是指客户端请求的数据在缓存中和数据库中都不存在，这样缓存不会生效，请求直接请求到数据库。常见的解决方案有两种：
- 缓存空对象
  - 优点：实现简单，维护方便
  - 缺点：
    - 额外的内存消耗
    - 可能造成短期的不一致
  
  ![缓存空对象示意图](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/redis/cache_through_1.png?raw=true)
- 布隆过滤器
  - 优点：内存空间占用少，没有多余的key
  - 缺点：
    - 实现复杂
    - 存在误判的可能
  
  ![布隆过滤器示意图](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/redis/cache_through_2.png?raw=true)


缓存雪崩是指在同一时段大量的缓存key同时失效或者redis服务宕机，导致大量请求达到数据库，带来巨大的压力。

解决方案：

- 给不同的key的ttl添加随机值
- 利用redis集群提高服务的可用性
- 给缓存业务添加降级限流策略
- 给业务添加多级缓存

## 缓存击穿

缓存击穿问题也叫热点key问题，就是一个被高并发并且缓存重建业务比较复杂的key突然失效了，无数的请求访问会在瞬间给数据库带来巨大的冲击。

常见的解决方案：
- 互斥锁
  
  ![互斥锁流程图]()
  
  
- 逻辑过期

  ![逻辑过期流程图]()

