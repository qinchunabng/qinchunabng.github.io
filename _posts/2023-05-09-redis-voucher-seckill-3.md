---
layout: post
title: 优惠券秒杀（三）—— 一人一单
categories: [Redis]
description: 一人一单
keywords: Redis, 秒杀
---


## 优惠券秒杀（二）—— 一人一单

需求：修改秒杀业务，要求同一个优惠券，一个用户只能下一单。

那么在下单时判断满足秒杀活动已经开始且库存充足的同时，还要再加一个判断条件：判断该用户是否已经下单。修改后下单流程如下：

![1653371854389.png](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/redis/1653371854389.png?raw=true)

实现代码：
```
@Override
public Result seckillVoucher(Long voucherId,Long userId) {
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
    // 5.一人一单逻辑
    // 5.1.用户id
    int count = query().eq("user_id", userId).eq("voucher_id", voucherId).count();
    // 5.2.判断是否存在
    if (count > 0) {
        // 用户已经购买过了
        return Result.fail("用户已经购买过一次！");
    }
    //6，扣减库存
    boolean success = seckillVoucherService.update()
            .setSql("stock= stock -1")
            .eq("voucher_id", voucherId).update().gt("stock",0); //where id = ? and stock > 0
    if (!success) {
        //扣减库存
        return Result.fail("库存不足！");
    }
    //7.创建订单
    VoucherOrder voucherOrder = new VoucherOrder();
    // 7.1.订单id
    long orderId = redisIdWorker.nextId("order");
    voucherOrder.setId(orderId);
    // 7.2.用户id
    Long userId = UserHolder.getUser().getId();
    voucherOrder.setUserId(userId);
    // 7.3.代金券id
    voucherOrder.setVoucherId(voucherId);
    save(voucherOrder);

    return Result.ok(orderId);

}
```

还是和之前一样，这段代码在并发请求下会出现线程安全的问题。同一个用户多个请求同时过来，都查询到没有下单，然后执行下单逻辑，就会出现一个用户下了多个订单。所以这里还是要加锁，但是乐观锁比适合更新数据库，而这里是插入数据，所以我们需要使用悲观锁。

我们可以封装一个createVoucherOrder方法，给createVoucherOrder方法加锁：
```
@Transactional
public synchronized Result createVoucherOrder(Long voucherId, Long userId) {
    // 5.一人一单逻辑
    // 5.1.用户id
    int count = query().eq("user_id", userId).eq("voucher_id", voucherId).count();
    // 5.2.判断是否存在
    if (count > 0) {
        // 用户已经购买过了
        return Result.fail("用户已经购买过一次！");
    }
    //6，扣减库存
    boolean success = seckillVoucherService.update()
            .setSql("stock= stock -1")
            .eq("voucher_id", voucherId).update().gt("stock",0); //where id = ? and stock > 0
    if (!success) {
        //扣减库存
        return Result.fail("库存不足！");
    }
    //7.创建订单
    VoucherOrder voucherOrder = new VoucherOrder();
    // 7.1.订单id
    long orderId = redisIdWorker.nextId("order");
    voucherOrder.setId(orderId);
    // 7.2.用户id
    Long userId = UserHolder.getUser().getId();
    voucherOrder.setUserId(userId);
    // 7.3.代金券id
    voucherOrder.setVoucherId(voucherId);
    save(voucherOrder);

    return Result.ok(orderId);
}
```
但是这样加锁，锁的粒度太大，会导致所有线程串行执行，从而使得TPS下降。所以代码可以进一步优化，使用userId.toString().intern()来作为锁对象，不同的用户使用不同的锁对象，降低锁的粒度。这里必须使用userId.toString().intern()，因为userId.toString()得到的是不同的对象，使用intern()是从字符串常量池中获取的对象，这样同一个用户就是同样的锁对象。优化后的代码：
```
@Transactional
public Result createVoucherOrder(Long voucherId, Long userId) {
    synchronized(userId.toString().intern()){
        // 5.一人一单逻辑
        // 5.1.用户id
        int count = query().eq("user_id", userId).eq("voucher_id", voucherId).count();
        // 5.2.判断是否存在
        if (count > 0) {
            // 用户已经购买过了
            return Result.fail("用户已经购买过一次！");
        }
        //6，扣减库存
        boolean success = seckillVoucherService.update()
                .setSql("stock= stock -1")
                .eq("voucher_id", voucherId).update().gt("stock",0); //where id = ? and stock > 0
        if (!success) {
            //扣减库存
            return Result.fail("库存不足！");
        }
        //7.创建订单
        VoucherOrder voucherOrder = new VoucherOrder();
        // 7.1.订单id
        long orderId = redisIdWorker.nextId("order");
        voucherOrder.setId(orderId);
        // 7.2.用户id
        Long userId = UserHolder.getUser().getId();
        voucherOrder.setUserId(userId);
        // 7.3.代金券id
        voucherOrder.setVoucherId(voucherId);
        save(voucherOrder);

        return Result.ok(orderId);
    }
}
```
但是以上代码还是有问题，因为锁是在事务提交之前释放的。如果此时有其他线程进来，仍然没有查到订单，还是出现重复下单的问题，所以锁应该加在事务之外。另外为了避免事务失效，不能通过this调用这个方法，应该通过代理对象调用。改造后的代码如下：
```
@Transactional
public Result createVoucherOrder(Long voucherId, Long userId) {
    // 5.一人一单逻辑
    // 5.1.用户id
    int count = query().eq("user_id", userId).eq("voucher_id", voucherId).count();
    // 5.2.判断是否存在
    if (count > 0) {
        // 用户已经购买过了
        return Result.fail("用户已经购买过一次！");
    }
    //6，扣减库存
    boolean success = seckillVoucherService.update()
            .setSql("stock= stock -1")
            .eq("voucher_id", voucherId).update().gt("stock",0); //where id = ? and stock > 0
    if (!success) {
        //扣减库存
        return Result.fail("库存不足！");
    }
    //7.创建订单
    VoucherOrder voucherOrder = new VoucherOrder();
    // 7.1.订单id
    long orderId = redisIdWorker.nextId("order");
    voucherOrder.setId(orderId);
    // 7.2.用户id
    Long userId = UserHolder.getUser().getId();
    voucherOrder.setUserId(userId);
    // 7.3.代金券id
    voucherOrder.setVoucherId(voucherId);
    save(voucherOrder);

    return Result.ok(orderId);
}

@Override
public Result seckillVoucher(Long voucherId,Long userId) {
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
    synchronized(userId.toString().intern()){
        //获取代理对象
        IVoucherOrderService proxy = (IVoucherOrderService) AopContext.currentProxy();
        return proxy.createVoucherOrder(voucherId, userId);
    }
}
```