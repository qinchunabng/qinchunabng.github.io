---
layout: post
title: 优惠券秒杀（四）—— 分布式锁
categories: [Redis]
description: 分布式锁
keywords: Redis, 秒杀
---

## 优惠券秒杀（四）—— 分布式锁

通过加锁可以解决单机情况下的一人一单的线程安全的问题，但是在集群模式下就不行了，因为JVM的锁只在当前JVM进程有效。例如现在秒杀服务部署了两个进程A和B在不同的服务器上，通过nginx反向代理做负载均衡。此时进程A内部同一个用户有线程1和线程2在进行秒杀下单，进程B内存也有线程3和线程4在秒杀下单。由于线程1和线程2的sync锁是同一个，会发生互斥。但是线程3和线程4在进程B中，不在同一个JVM，那么锁对象肯定不是同一个，所以线程3和线程4不会和线程1、线程2互斥，就出现重复下单。整个流程如下：

![1653374044740](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/redis/1653374044740.png?raw=true)

### 分布式锁

出现这种问题的原因是Java的锁只能在当前JVM进程有效，需要一个跨进程的锁实现互斥，于是就有个分布式锁。分布式锁实现的核心思想就是让大家使用同一把锁，只要大家使用的是同一把锁，那么我们就能把线程锁住，让程序串行执行。在使用分布式锁流程如下：

![1653374296906](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/redis/1653374296906.png?raw=true)

那么分布式锁应该满足什么条件呢？

- 可见性：多个线程都能看到相同的结果，注意这里说到可见性并不是并发编程的内存可见性，只是说多个进程之间都能感知到变化的意思。
- 互斥：互斥是分布式锁的最基本的条件，使得程序串行执行。
- 高可用：分布式在大多数时候都可用，不能因为分布式锁的问题影响到业务。
- 高性能：由于加锁本身就能让性能下降，所以对于分布式锁要有较高的加锁性能和释放锁性能
- 安全性：分布式锁考虑异常情况，如果出现异常不会出现死锁等问题。

常见的分布式锁实现方案有三种：

- MySQL：MySQL本身就带有锁机制，但是由于MySQL本身性能一般，所以一般不使用MySQL作为分布式锁的实现。
- Redis: Redis作为分布式锁是非常常见的方式，利用setnx这个命令，如果操作成功表示加锁成功，其他线程就会操作失败，就表示加锁失败。利用这套逻辑来实现分布式锁。
- ZooKeeper: 利用Zookeeper创建节点的唯一性和有序性来实现分布式锁，多个线程同时创建同一个节点只有一个能够创建成功。Zookeeper来实现分布式锁也是一种比较常见的方式。

三种实现方式的对别：

||MySQL|Redis|Zookeeper|
|:-:|:-:|:-:|:-:|
|互斥|利用MySQL本身的互斥锁机制|利用setnx这样的互斥命令|利用节点为唯一性和有序性实现互斥|
|高可用|好|好|好|
|高性能|一般|好|一般|
|安全性|断开连接，自动释放锁|利用锁超时机制，到期释放锁|临时节点，断开连接自动释放锁|

#### Redis分布式锁的实现

我们这里主要讲解Redis分布式的实现。Redis分布式锁的需要实现的两个基本方法：
- 获取锁：
  - 互斥：确保只能有一个线程获取锁。通过SETNX实现，多个线程同时通过SETNX设置只有一个线程可以成功。
    ```
    # 添加锁
    SETNX lock thread1
    # 设置超时时间
    EXPIRE lock 10
    ```
    但是SETNX和EXPIRE是两个操作，无法保证原子性。所以可以用一个命令代替：
    ```
    SET lock thread1 EX 10 NX
    ```
    SET命令EX参数设置过期时间，NX参数只允许key不存在时才能设置成功。

  - 非阻塞：尝试一次，成功返回true，失败返回false。

- 释放锁：
  - 手动释放。直接删除对应的key：
    ```
    DEL key
    ```
  - 超时释放。获取锁时设置超时时间，防止因为异常导致锁无法释放出现死锁。

#### 基于Redis实现分布式锁的初级版本

定义分布式锁的接口，实现这个接口，利用Redis实现分布式锁的功能。
```
public interface ILock{

    /**
    * 尝试获取锁
    * @param timeoutSec 所持有超时时间，过期后自动删除
    * @return true代表加锁成功，false代表加锁失败
    */
    boolean tryLock(long timeoutSec);

    /**
    * 释放锁
    */
    void unlock();
}
```

基于Redis分布式锁实现代码：
```
public class SimpleRedisLock implements ILock {

  /**
  * 锁名称
  */
  private String name;
  private StringRedisTemplate stringRedisTempalte;

  private static final String KEY_PREFIX = "lock:";

  public SimpleRedisLock(String name, StringRedisTemplate stringRedisTempalte){
    this.name=name;
    this.stringRedisTempalte=stringRedisTempalte;
  }

  /**
  * 加锁操作
  */
  @Override
  public boolean tryLock(long timeoutSec){
    //获取线程表示
    long threadId = Thread.currentThread().getId();
    //获取锁
    Boolean success = stringRedisTemplate.opsForValue().setIfAbsent(KEY_PREFIX + name, threadId, timeoutSec, TimeUnit.SECONDS);
    //防止自动拆箱导致空指针异常
    return Boolean.TURE.equals(success);
  }

  /**
  * 释放锁
  */
  @Overrider
  public unlock(){
    stringSimpleTemplate.delete(KEY_PREFIX + name);
  }
}
```

替换优惠券秒杀下单加锁操作：
```
//创建锁对象
ILock lock = new SimpleRedisLock("order:" + userId, stringRedisLock);
//获取锁
boolean isLock = lock.tryLock(5);
//判断加锁是否成功
if(!isLock){
  //获取锁失败，返回错误信息或者重试
  return Result.fail("不允许重复下单");
}
try{
  //获取代理对象
  IVoucherOrderService proxy = (IVoucherOrderService) AopContext.currentProxy();
  return proxy.createVoucherOrder(voucherId, userId)
}finally{
  lock.unlock();
}
```

#### Redis分布式锁误删问题

考虑如下场景：线程1先获取分布式锁，执行业务，但是由于某种原因，业务代码阻塞导致分布式锁超时，锁被删除。这个时候线程2获取分布式锁，执行业务,过了一会儿线程1业务执行完成，直接删除锁，就导致线程2的锁被误删。然后又有线程3发起请求，并且获取分布式锁成功。这个时间就出现同时两个线程获取到分布式锁的情况，就有可能导致数据安全的问题。整个流程如下图所示：

![分布式锁误删](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/redis/del_wrong_lock.png?raw=true)

如何解决分布式锁误删问题呢？在获取的时候把当前线程的标识存进群，然后释放锁的时候，先判断锁标识与当前线程是否一致，一致才删除释放锁。优化的释放锁的流程如下：

![分布式锁误删](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/redis/del_wrong_lock_1.png?raw=true)

改进Redis分布式锁实现：
1. 获取锁时，存入线程唯一标识（可以用UUID标识）
2. 在释放锁时先获取锁种线程标识，判断是否与当前线程一致
   - 如果一致则释放锁
   - 如果不一致则不释放锁

改进代码：
```
private static final String ID_PREFIX = UUID.randomUUID().toString().replaceAll("-","") + "-;

* 加锁操作
*/
@Override
public boolean tryLock(long timeoutSec){
  //获取线程表示
  long threadId = ID_PREFIX + Thread.currentThread().getId();
  //获取锁
  Boolean success = stringRedisTemplate.opsForValue().setIfAbsent(KEY_PREFIX + name, threadId, timeoutSec, TimeUnit.SECONDS);
  //防止自动拆箱导致空指针异常
  return Boolean.TURE.equals(success);
}

/**
* 释放锁
*/
@Overrider
public unlock(){
  //获取线程标识
  long threadId = ID_PREFIX + Thread.currentThread().getId();
  //获取锁种的线程标识
  String id = stringRedisTemplate.opsForValue().get(KEY_PREFIX + name);
  //判断线程标识是否一致
  if(threadId.equals(id)){
    //释放锁
    stringSimpleTemplate.delete(KEY_PREFIX + name);
  }
}
```

#### 分布式锁的原子性

看下面一个特殊场景：
![分布式锁原子性](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/redis/distribution_lock_atomic.png?raw=true)

线程1先获取分布式执行自己的业务，执行完之后，获取分布式锁中的锁标识与当前线程标识进行比较，判断锁标识一致后，在删除锁之前，由于GC导致当前线程阻塞。最后分布式锁由于超时被自动删除，这个时候线程2获取分布式锁成功，执行自己的业务。此时线程1从阻塞中恢复，由于之前已经判断过锁标识一致，直接执行删除锁的代码，就将线程2的锁删除了。然后线程3获取到分布式锁执行自己的业务，这个就存在线程1和线程2同时在执行，就会导致数据安全的问题。出现这个问题的原因是，判断锁是否一致和删除锁是两个操作，无法保证原子性。