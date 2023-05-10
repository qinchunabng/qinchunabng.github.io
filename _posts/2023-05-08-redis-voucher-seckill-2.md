---
layout: post
title: 优惠券秒杀（二）—— 秒杀下单
categories: [Redis]
description: 秒杀下单
keywords: Redis, 秒杀,
---

## 优惠券秒杀（二）—— 秒杀下单

每个店铺都可以发布优惠券，分为平价券和特价券。平价券可以任何购买，而特价券需要秒杀抢购。涉及到两张表tb_voucher和tb_voucher_order：
- tb_voucher：优惠券的基本信息，优惠金额、使用规则等。建表语句如下：
  ```
  CREATE TABLE tb_voucher_order  (
  id bigint(20) NOT NULL COMMENT '主键',
  user_id bigint(20) UNSIGNED NOT NULL COMMENT '下单的用户id',
  voucher_id bigint(20) UNSIGNED NOT NULL COMMENT '购买的代金券id',
  pay_type tinyint(1) UNSIGNED NOT NULL DEFAULT 1 COMMENT '支付方式 1：余额支付；2：支付宝；3：微信',
  status tinyint(1) UNSIGNED NOT NULL DEFAULT 1 COMMENT '订单状态，1：未支付；2：已支付；3：已核销；4：已取消；5：退款中；6：已退款',
  create_time timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '下单时间',
  pay_time timestamp NULL DEFAULT NULL COMMENT '支付时间',
  use_time timestamp NULL DEFAULT NULL COMMENT '核销时间',
  refund_time timestamp NULL DEFAULT NULL COMMENT '退款时间',
  update_time timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (id) USING BTREE
  ) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Compact;
  ```

- tb_seckill_voucher: 优化券的库存、开始抢购时间、结束抢购时间。特价优惠券才需要这些信息。建表语句如下：
  ```
  CREATE TABLE `tb_voucher_order`  (
  `id` bigint(20) NOT NULL COMMENT '主键',
  `user_id` bigint(20) UNSIGNED NOT NULL COMMENT '下单的用户id',
  `voucher_id` bigint(20) UNSIGNED NOT NULL COMMENT '购买的代金券id',
  `pay_type` tinyint(1) UNSIGNED NOT NULL DEFAULT 1 COMMENT '支付方式 1：余额支付；2：支付宝；3：微信',
  `status` tinyint(1) UNSIGNED NOT NULL DEFAULT 1 COMMENT '订单状态，1：未支付；2：已支付；3：已核销；4：已取消；5：退款中；6：已退款',
  `create_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '下单时间',
  `pay_time` timestamp NULL DEFAULT NULL COMMENT '支付时间',
  `use_time` timestamp NULL DEFAULT NULL COMMENT '核销时间',
  `refund_time` timestamp NULL DEFAULT NULL COMMENT '退款时间',
  `update_time` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  PRIMARY KEY (`id`) USING BTREE
  ) ENGINE = InnoDB CHARACTER SET = utf8mb4 COLLATE = utf8mb4_general_ci ROW_FORMAT = Compact;
  ```

### 实现秒杀下单

用户下单时，首先要查询优惠券信息，判断是否满足几个条件：
- 判断秒杀是否开始或结束，如果未开始或者已经结束则无法下单。
- 判断库存是否充足，不足则无法下单。

整个下单流程如下：

![秒杀下单流程](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/redis/1653366238564.png?raw=true)

秒杀核心实现代码：
```
@Override
public Result seckillVoucher(Long voucherId) {
    // 1.查询优惠券
    SeckillVoucher voucher = seckillVoucherService.getById(voucherId);
    // 2.判断秒杀是否开始
    if (voucher.getBeginTime().isAfter(LocalDateTime.now())) {
        // 尚未开始
        return Result.fail("秒杀尚未开始！");
    }
    // 3.判断秒杀是否已经结束
    if (voucher.getEndTime().isBefore(LocalDateTime.now())) {
        // 尚未开始
        return Result.fail("秒杀已经结束！");
    }
    // 4.判断库存是否充足
    if (voucher.getStock() < 1) {
        // 库存不足
        return Result.fail("库存不足！");
    }
    //5，扣减库存
    boolean success = seckillVoucherService.update()
            .setSql("stock= stock -1")
            .eq("voucher_id", voucherId).update();
    if (!success) {
        //扣减库存
        return Result.fail("库存不足！");
    }
    //6.创建订单
    VoucherOrder voucherOrder = new VoucherOrder();
    // 6.1.订单id
    long orderId = redisIdWorker.nextId("order");
    voucherOrder.setId(orderId);
    // 6.2.用户id
    Long userId = UserHolder.getUser().getId();
    voucherOrder.setUserId(userId);
    // 6.3.代金券id
    voucherOrder.setVoucherId(voucherId);
    save(voucherOrder);

    return Result.ok(orderId);

}
```

### 库存超卖问题

以上代码实际是有问题的，如果在高并发场景下，会出现订单比库存多，即库存超卖的问题。那超卖是如何发生的，我看下单逻辑如下代码段：
```
 // 4.判断库存是否充足
if (voucher.getStock() < 1) {
    // 库存不足
    return Result.fail("库存不足！");
}
//5，扣减库存
boolean success = seckillVoucherService.update()
        .setSql("stock= stock -1")
        .eq("voucher_id", voucherId).update();
if (!success) {
    //扣减库存
    return Result.fail("库存不足！");
}
```
在这段代码中，我们先查库存之后，判断不小于1之后才去扣减库存。在高并发的场景中，可能出现多个请求同时进来，假设当时库存为1，都查询到库存1，于是多个请求又都去扣减库存，就会出现库存为负的超卖问题。

![库存超卖流程图](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/redis/1653368335155.png?raw=true)

库存超卖的问题就是典型的线程安全的问题，解决线程安全的问题的常见方案就是加锁。而对于加锁，有两种解决方案：

- 悲观锁。即认为线程安全问题一定会发生，所以在操作数据之前要先获取锁，确保线程串行执行。例如Synchonized、Lock都属于悲观锁。同时悲观锁又可以细分为公平锁、非公平锁、可重入锁等等。
  
- 乐观锁。即认为线程安全问题不一定会发生，因此不加锁，只是在更新数据的时候去判断其他线程有没有对数据做了修改。如果没有修改则认为是安全，自己才更新数据。如果已经被其他线程修改说明发生了线程安全问题，发起重试或者抛出异常。乐观锁一般都会有一个版本号，每次操作数据版本号都会+1，再提交数据时会校验版本是否相同，如果相同就意味着操作过程种没有其他人对数据进行修改，可以提交数据。否则说明操作过程种有其他人修改了数据，数据是不安全的，就需要进行重试。

在这里我可以采用乐观锁的方式解决库存超卖问题，给库存加一个版本号，通过版本号来实现乐观锁，整个流程如下：

![乐观锁扣减库存](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/redis/1653369268550.png?raw=true)

我们可以将库存作为版本号使用，因为如果在下单过程种，库存只要发生变化，就说有其他人下单，库存数据就不安全。但是如果每次提交订单时，判断库存是否变化，失败率太高，因为在秒杀过程中，并发会很高，会有个线程都在扣减库存。而我们关系的是只要不出现库存为负超卖问题就行，所以在提交订单扣减库存时，只要判断库存>0就行。扣减库存的代码修改为：
```
boolean success = seckillVoucherService.update()
            .setSql("stock= stock -1")
            .eq("voucher_id", voucherId).update().gt("stock",0); //where id = ? and stock > 0
```

### 小结

超卖这样的线程安全问题，解决方案有哪些？
1. 悲观锁：添加同步锁，让线程串行执行。
   - 优点：简单粗暴
   - 缺点：性能一般
2. 乐观锁：不加锁，在更新时判断是否有其他线程在修改。
   - 优点：性能好
   - 缺点：存在成功率低的问题
