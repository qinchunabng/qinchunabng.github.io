---
layout: post
title: 优惠券秒杀（-）—— 全局唯一ID
categories: [Redis]
description: 全局唯一ID
keywords: Redis, 秒杀, 全局唯一ID
---

## 优惠券秒杀（-）—— 全局唯一ID

秒杀是电商中比较常见的业务场景，其特点就是商品价格低，数量有限，就导致会有很多用户来抢，就会出现瞬间请求量剧增。如果这些请求直接到达数据库，会导致数据库超过自己的性能瓶颈而出现CPU内存飙升，而无法处理请求。所以这种业务场景使用redis来处理再合适不过，redis具有优秀的性能，能抗住大量数据请求。



### 全局唯一ID

在优惠券秒杀项目中，抢到优惠券需要生成订单保存到tb_voucher_order表中，如果订单表使用数据库自增ID就会有存在一些问题：
- id规律太明显
- 受单表数据量的限制

场景分析一：如果id规则太明显，就很容易被人猜到一些敏感信息，比如一定卖出多少订单，这明显不合适。

场景分析二：随着商城规模越来越大，MySQL的单表容量不易超过500w，数据量过大之后，需要进行分库分表，但拆分表之后，它们逻辑上还是一张表，所以id不能是一样的，所以我们要保证ID的唯一性，这个时候就需要使用到全局唯一ID生成器。

**全局唯一ID生成器**是一种在分布式系统下用来生成全局唯一ID的工具，一般要满足下列特性：

- 唯一性。唯一性很好理解，生成的ID要全局唯一。
- 高可用。生成的全局唯一ID方式是要高可用的，不能因为某些原因导致无法生成ID，从而影响业务。
- 高性能。生成唯一ID的方法要性能好，因为很多业务都需要用到这个方法，所以如果性能不好，也会导致其他的功能受到影响。
- 递增性。生成的ID要保持单调递增，原因是因为数据库的主键数据结构一般都是B+树，索引值单调递增可能防止B+树节点的合并和分裂，减少因为节点合并和分裂带来的性能开销。
- 安全性。前面我们提供如果用数据的自增ID的方式可能会导致敏感信息的泄漏，让别人猜到每天的订单数量，所以ID生成规则需要避免这种问题。

为了满足以上条件，我们不直接使用Redis的自增数值，而是拼接了一些其他的信息：
![1653363172079.png](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/redis/1653363172079.png?raw=true)

ID组成部分：
- 符号位：1bit，永远为0
- 时间戳：31bit，以秒为单位，可以使用69年
- 序列号：32bit，秒内的计时器，支持每秒产生2^32个不同的ID

### Redis实现全局唯一ID

时间戳为某个时间点到当前时间的秒数，自增序列使用Redis的incr命令实现，保证原子性。自增序列号按照业务key已经当前日期组成：`icr:{keyPre}:{yyyy}:{MM}:{dd}`，每天都产生新的序列这样的好处是可以按照业务分类，且可以统计每天的订单数量。获取当前时间后，位运算左移32位，使低32位为0，高32位为当前时间戳，并且由于时间戳为正整数，所以第一位为0。获取序列号后通过或运算，将时间戳和序列号组装起来。实现代码如下：
```
/**
 * ID生成器
 *
 * @author qcb
 * @date 2022/09/17 18:48.
 */
@Component
public class RedisIdWorker {

    /**
     * 开始时间戳
     */
    private static final long BEGIN_TIMESTAMP = 1640995200L;

    /**
     * 序列号位数
     */
    private static final int SEQUENCE_BIT_COUNT = 32;

    @Autowired
    private RedisTemplate redisTemplate;

    /**
     * 生成ID
     * ID分为三个部分
     * 0 - 0000000 00000000 00000000 - 00000000 00000000 00000000
     * 符号位 - 时间戳（31bit） - 序列号（32bit）
     * @param keyPrefix
     * @return
     */
    public long nextId(String keyPrefix){
        //1.生成时间戳
        LocalDateTime now = LocalDateTime.now();
        long nowSecond = now.toEpochSecond(ZoneOffset.UTC);
        long timestamp = nowSecond - BEGIN_TIMESTAMP;

        //2.生成序列号
        //2.1 获取当前的日期
        String date = now.format(DateTimeFormatter.ofPattern("yyyy:MM:dd"));
        //2.2 自增长
        long count = redisTemplate.opsForValue().increment("icr:" + keyPrefix + ":" + date);
        //3.拼接返回
        return timestamp << SEQUENCE_BIT_COUNT | count;
    }

    public static void main(String[] args) {
        LocalDateTime time = LocalDateTime.of(2022,1,1,0,0,0);
        long second = time.toEpochSecond(ZoneOffset.UTC);
        System.out.println("Second = " + second);
    }
}
```
