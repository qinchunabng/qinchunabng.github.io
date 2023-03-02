---
layout: post
title: MySQL InnoDB之加锁的读
categories: [MySQL]
description: MySQL InnoDB之加锁的读
keywords: 数据库, MySQL, InnoDB, 锁
---

如果你需要在数据库中执行一个查询操作，然后根据查询结果来插入或者更新这些数据，由于普通的SELECT语句并不会加锁，所以其他事务此时可以插入和修改这些数据，就可能会查询数据安全的问题，产生数据不一致。InnoDB提供两种方式给读操作加锁，来避免这种情况：
- SELECT...FOR SHARE
  这个语句会给读取的数据加共享锁，其他会话可以这个事务没有提交前只能读取数据不能修改数据。如果有其他事务在修改读取的数据并且还没有提交，你的查询操作会等待知道事务提交，然后获取最新的数据。
  > **注意**
  > SELECT...FOR SHARE是SELECT...LOCK IN SHARE MODE的替代品，保留LOCK IN SHARE MODE是为了向后兼容，两个语句是相等的。

  在MySQL 8.0.22之前SELECT...FOR SHARE需要SELECT权限和DELETE,LOCK TABLES,UPDATE三个中最少一个的权限，从MySQL 8.0.22开始只需要SELECT权限。

- SELECT...FOR UPDATE
  这个语句在查询过程中会锁住数据行和相关的索引条目，类似UPDATE操作。其他事务的更新操作，SELECT...FOR SHARE以及有些事务的隔离级别（主要是SERIALIZABLE）的读操作都会阻塞。一致性读会忽略记录上的任何锁。（旧版本的数据记录不会被阻塞，是因为这些数据是通过undo日志重建的。）
  SELECT...FOR UPDATE需要SELECT权限和DELETE,LOCK TABLES和UPDATE权限中的至少一种权限。

FOR SHARE和FOR UPDATE查询中加的锁会在事务提交或者回滚时释放。

> **注意**
> 加锁的读只有在autocommit关闭的时候才有效（通过START TRANSACTION开启事务或者这只autocommit=0）。

外部SQL中的读锁不会锁住内部子查询的数据，除非内部子查询中也使用了读锁。例如，下面的语句不会锁住表t2:
```
SELECT * FROM t1 WHERE c1 = (SELECT c1 FROM t2) FOR UPDATE;
```
如果要锁住子查询，在查询中也要加FOR UPDATE或FOR SHARE:
```
SELECT * FROM t1 WHERE c1 = (SELECT c1 FROM t2 FOR UPDATE) FOR UPDATE;
```

### 读锁的使用案例
假设现在你需要在child表中插入一条新的数据，并且确保parent表中有一条对应的关联数。首先你需要查询parent表，判断parent表中对应的数据存在。那么这个你可以安全的往child表中插入数据吗？答案是否定的，因为其他的会话可能在你的SELECT和INSERT之间删除了parent表的这条数据。

为了避免这种情况的发生，你可以给SELECT语句加上FOR SHARE:
```
SELECT * FROM parent WHERE NAME = 'Jones' FOR SHARE;
```
在FOR SHARE查询返回NAME='Jones'记录之后，你可以安全的插入数据到chind表。任何其他的事务想要获取parent中NAME='Jones'这行数据的独占锁都会等待直到你的事务结束，这个过程中，所有的数据都是一致性的状态。

另外一个案例，假设child_codes表中有一个integer类型的counter字段，用来统计child表的数据总数。这个时候不能使用一致性读，也不能使用FOR SHARE共享读，因为两个会话在同一时刻执行会看到相关counter值，并且两个事务如果插入相同数据数据会出现重复key的错误。

FOR SHARE在这里不适用是因为如果两个用户在同一时刻读取counter值，在尝试更新counter值的时候至少一个事务会出现死锁。

实现读取并增加counter计数，首先应该通过SELECT...FOR UPDATE读取数据，然后再更新counter值。例如：
```
SELECT counter_field FROM child_codes FOR UPDATE;
UPDATE child_codes SET counter_field = counter_field + 1;
```

SELECT...FOR UPDATE会去读取最新的数据，并且给读取的数据行加独占锁，所以它设置的锁是UPDATE语句一样的。

以上的例子只是演示SELECT...FOR UPDATE是如何工作的，实际上在MySQL中可以通过一条SQL完成以上操作：
```
UPDATE child_codes SET counter_field = LAST_INSERT_ID(counter_field + 1);
SELECT LAST_INSERT_ID();
```
  