---
layout: post
title: MySQL InnoDB中的一致性非阻塞读
categories: [MySQL]
description: MySQL InnoDB中的一致性非阻塞读
keywords: 数据库, MySQL, InnoDB, 一致性
---

一致性读就是InnoDB利用多版本并发控制（MVCC）来查询某个时间点前的数据快照，查询只能查到这个时间点之前提交的数据，这个时间点之后提交和未提交的数据看不到。但是查询可以看到同一个事务中前面的改变的数据，但是也可能同时看到旧版本的数据。

如果事务的隔离级别时REPEATABLE READ(默认)，所有同一个事务中的一致性读都是第一个读取的数据快照。如果要查询最新的数据，需要提交事务然后重新发起查询。如果是READ COMMITTED，事务中每次读都是读取的最新数据快照。

在REPEATABLE READ和READ COMMITTED隔离级别中一致性读是默认的，一致性读不会加任何锁，所以一致性读发生的时候其他事务可以随意修改数据。

假设当前使用的是REPEATABLE READ的默认事务隔离级别，当发起一个SELECT查询，InnoDB会根据当前事务发生的时间点获取一个数据快照，如果其他事务在这个时间点之后执行了删除操作并且提交了事务，当前事务时查不到数据被删除，插入和更新操作也是如此。

> **注意**
>
> 数据库的快照读只适用与SELECT语句，不适用与DML语句。如果你插入或者修改一些数据然后提交了事务，另外一个REPEATABLE READ隔离级别的事务可以影响到这些刚提交的数据行，即使当前事务中无法查询到。如果一个事务更新或删除了其他事务提交的数据，这些数据变化在当前事务也能查询到。下面通过一个例子演示这种情况：
> ```
> SELECT COUNT(c1) FROM t1 WHERE c1 = 'xyz';
> -- Returns 0: 没有匹配的行。
> DELETE FROM t1 WHERE c1 = 'xyz';
> -- 删除几行被其他事务提交的数据。
> 
> SELECT COUNT(c2) FROM t1 WHERE c2 = 'abc';
> -- Returns 0: 没有匹配的行.
> UPDATE t1 SET c2 = 'cba' WHERE c2 = 'abc';
> -- 影响10行: 另外一个事务刚刚提交10行c2='abc'的数据。
> SELECT COUNT(c2) FROM t1 WHERE c2 = 'cba';
> -- 返回10: 这个事务可以看到刚刚更新的行。
> ```

你可以通过提交事务重新发起SELECT或者START TRANSACTION WITH CONSISTENT SNAPSHOT向前移动可见数据的时间点。这种机制就是多版本并发控制，检查MVCC。

下面这个例子中，会话A只有在会话B提交了事务并且会话A也提交了才能查看会话B插入的数据：
```
             Session A              Session B

           SET autocommit=0;      SET autocommit=0;
time
|          SELECT * FROM t;
|          empty set
|                                 INSERT INTO t VALUES (1, 2);
|
v          SELECT * FROM t;
           empty set
                                  COMMIT;

           SELECT * FROM t;
           empty set

           COMMIT;

           SELECT * FROM t;
           ---------------------
           |    1    |    2    |
           ---------------------
```

如果你查询到最新提交的数据，可以修改事务隔离级别为READ COMMITTED或者通过加锁的读：
```
SELECT * FROM t FOR SHARE;
```
FOR SHARE会查询最新的数据并且加锁直到事务结束。

一些DDL语句不会使用一致性读：
- DROP TABLE不会使用一致性读，因为MySQL不能查询一张已经被删除了的表。
- ALTER TABLE不会使用一致性读， 因为ALTER TABLE操作会创建一个临时表取代原表，并在将原表的数据复制过来之后删除原表。

INSERT INTO ... SELECT，UPDATE ... (SELECT)和CREATE TABLE ... SELECT这三种带SELECT查询且没有加FOR UPDATE和FOR SHARE的语句，读取方式又不一样：
- 默认情况，InnoDB会使用更强的锁，SELECT操作和READ COMMITTED隔离级别的一样，同一个事务中每次查询都是读取的最新数据快照。
- 如果执行无锁的读，需要设置事务隔离级别为READ UNCOMMITTED或者READ COMMITTED防止给读取的数据行加锁。