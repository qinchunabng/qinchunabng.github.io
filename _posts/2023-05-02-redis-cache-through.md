---
layout: post
title: 缓存穿透的解决方法
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