---
layout: post
title: MySQL InnoDB中不同SQL语句的锁
categories: [MySQL]
description: InnoDB的不同SQL加的锁
keywords: 数据库, MySQL, InnoDB,
---

在MySQL的InnoDB中，读锁、UPDATE操作、DELETE操作一般会给SQL执行过程中所有扫描的到索引记录加上记录锁，无论是否满足WHERE条件。加上锁一般都是next-key锁，防止其他事务在执行过程中在数据间隙中插入数据。但是间隙锁可以通过调整事务隔离级别的方式来禁用，在[MySQL InnoDB的锁这篇文章讲过](https://qinchunabng.github.io/2023/02/19/mysql-innodb-locking/)。

如果查询中用的是二级索引并且设置的是独占锁，InnoDB同时也会查询对应聚集索引并且设置锁。

如果没有SQL语句没有索引可用，MySQL将扫描全表，并且表中每一行数据将被锁住，其他的事务的所有插入操作将会被阻塞。索引查询中创建合适的索引是很有必要的，避免全表扫描。

对于不同的SQL语句，InnoDB的加锁规则如下：

- SELECT...FROM操作是一致性读，读取的是数据快照，并且不会加锁，除非设置事务的隔离级别为SERIALIZABLE。在SERIALIZABLE隔离级别下，查询会给扫描到索引记录设置共享next-key锁，但是对于唯一的索引查询操作只会加记录锁。

- SELECT...FOR UPDATE和SELECT...FOR SHARE语句使用的是唯一索引作为查询条件，对于不满查询条件的数据将会释放锁。然后在一些特殊情况下，不满足条件的数据行的锁不会被释放，因为再执行过程中，结果集和它原始数据之间的关系可能会丢失。比如，在UNION查询中，查询的出现的结果在判断是否满足条件前会被插入到一个临时表。在这种情况下，临时表的数据和原始的数据之间的关系会丢失，所以锁会到查询执行结束之后才会释放。
  
- 对于加锁的读(SELECT...FOR SHARE和SELECT...FOR UPDATE)，UPDATE和DELETE的操作语句，加什么锁取决于语句是否使用的唯一索引作为单个查询条件或者范围查询条件：
  - 对于用唯一索引为单个查询条件，InnoDB只会锁住单独行记录，不会给索引记录前的间隙加间隙锁。
  - 对于其他查询条件，或者使用非唯一索引的查询，InnoDB会锁定扫描的范围的数据，使用间隙锁或者next-key锁防止其他会话插入数据。
  
- 使用SELECT...FOR UPDATE查询语句扫描到的索引记录，会加锁阻塞其他会话执行SELECT...FOR SHARE，或者SERIALIZABLE事务隔离级别中普通读取操作（因为SERIALIZABLE会给普通读转换为SELECT...FOR SHARE，在[MySQL InnoDB事务模型之事务隔离级别](https://qinchunabng.github.io/2023/02/23/mysql-innodb-transaction-model-isolation-level/)这篇文章中介绍过）。另外一致性读会忽略查询记录上的任何锁。

- UPDATE...WHERE...语句会给查询条件扫描到的记录加Next-Key锁，但是对于唯一索引作为查询条件的查询单行数据，只会锁定对应的行记录。
  
- 当UPDATE语句修改的是聚集索引，受到影响的二级索引记录会被加上隐式锁。UPDATE操作中，在插入耳机索引记录前检查重复数据时，也会给受到影响的二级索引加共享锁。

- DELETE FROM ... WHERE ...语句会给查询过程中扫描到的行记录加上独占的Next-Key锁，但是如果查询条件是精确匹配的唯一索引，只会锁定对应的行数据。

- INSERT操作会给插入的行设置独占锁。这个锁是索引记录锁，不是一个Next-Key锁，所以其他会话在插入数据行前的间隙中插入数据不会被阻塞。
  
  在插入数据行之前，会加插入意向锁的间隙锁。这种锁标识有数据需要插入，但是多个事务插入同样的间隙中不同行不需要相互等待。假如现在有个表有值为4和7的索引记录数据，两个不同的事务分别准备插入值为5和6的数据行，在获取插入数据的独占锁之前，都会给4和7之前间隙加插入意向锁，但是两个事务不会相互阻塞，因为插入数据行是不同的。

  如果重复数据检查报错，会给重复的索引记录加上共享锁。这种情况下可能会产生死锁，因为多个事务插入同一行数据时，其他事务可能已经设置了独占锁。这个可能发生在另外一个会话删除数据时。下面通过一个例子演示一下，假设现在有一张表t1:
  ```
  CREATE TABLE t1 (i INT, PRIMARY KEY (i)) ENGINE = InnoDB;
  ```
  现在有三个数据库会话执行按照顺序执行三个不同的操作：

  Session 1:
  ```
  START TRANSACTION;
  INSERT INTO t1 VALUES(1);
  ```
  Session 2:
  ```
  START TRANSACTION;
  INSERT INTO t1 VALUES(1);
  ```
  Session 3:
  ```
  START TRANSACTION;
  INSERT INTO t1 VALUES(1);
  ```
  Session 1：
  ```
  ROLLBACK;
  ```
  会话一的操作给插入数据设置了一个独占锁，会话2个会话3的插入会出现值重复的异常，然后他们都会取获取插入行的共享锁。当会话1的事务回滚之后，会话2和3获取共享锁成功，但是他们都无法获取到插入数据行的独占锁，因为两个会话都有共享锁。

  如果表中已经有值为1的数据，三个事务按照顺序执行下面的操作，同样的情况也可能发生：

  Session 1:

  ```
  START TRANSACTION;
  DELETE FROM t1 WHERE i = 1;
  ```
  Session 2:
  ```
  START TRANSACTION;
  INSERT INTO t1 VALUES(1);
  ```
  Session 3:
  ```
  START TRANSACTION;
  INSERT INTO t1 VALUES(1);
  ```
  Session 1:
  ```
  COMMIT;
  ```
  第一个操作会给数据行加上独占锁，会话2和3的操作会出现重复值异常，然后都会取数据行的共享锁，会话1的事务提交之后释放独占锁，会话2和3获取共享锁成功。这个时候就会出现思索，因为会话2和3都有数行的共享锁，导致都无法成功获取到独占锁。

- INSERT...ON DUPLICATE KEY UPDATE与普通的INSERT不同的时，当发送重复值错误时，会加独占锁而不是共享锁。会给重复主键索引值加独占的记录锁，给重复的唯一索引值加独占的next-key锁。

- REPLACE语句在没有唯一索引值冲突的时候，与INSERT是一样的。有冲突的时候，被替换的数据行会加上独占的next-key锁。

- INSERT INTO T SELECT...FROM S WHERE...会给插入每一行设置独占的记录所（无间隙锁）。如果事务隔离级别是READ COMMITTED,InnoDB查询S表时会使用一致性读（不加锁）。不然InnoDB会给S表的数据加上共享的next-key锁。在第二种情况下，InnoDB必须加锁，因为基于Statement的二进制日志在数据回滚的时候，每条语句执行过程应该是原始操作是一致的。
  
  CREATE TABLE...SELECT...执行SELECT查询使用的是next-key锁或者一致性读，就和INSERT...SELECT一样。
  
  在 REPLACE INTO t SELECT ... FROM s WHERE ... 或者 UPDATE t ... WHERE col IN (SELECT ... FROM s ...)这样的语句中，InnoDB给S表中数据行设置共享的next-key锁。

- 如果一个列设置为AUTO_INCREMENT，InnoDB会在对应索引列的末端加独占锁。设置innodb_autoinc_lock_mode=0,InnoDB在访问自增计数值的时候，会使用一个特殊的AUTO-INC表锁，这个锁获取到后会持有到SQL执结束（不是整个事务执行结束）。其他事务在AUTO-INC锁被锁住之后，无法执行插入操作。当设置innodb_autoinc_lock_mode=1执行批量插入也是一样的。innodb_autoinc_lock_mode=2时，不会使用AUTO-INC锁。
  
- 如果一个表设置有外键约束，任何需要检查约束条件的插入、更新和删除操作都会检查约束约束条件的时候给检查的记录设置行级的共享锁，并且即使检查约束失败也会加锁。

- LOCK TABLES设置表锁，但是表锁是InnoDB数据引擎层上的MySQL层设置的。如果innodb_table_locks = 1 (默认值) 和 autocommit = 0,InnoDB知道设置了表锁，并且MySQL层也知道行级锁的设置。否则InnoDB的死锁检测机制无法检查表锁导致的死锁。另外由于MySQL也不知道行级锁的存在，也会导致另外一个会话加了行级锁的同时也能获取表级锁。
- 如果设置innodb_table_locks=1 (默认值)，LOCK TABLES会获取两个锁。除了MySQL层的表级锁，也会获取InnoDB的表级锁。通过设置innodb_table_locks=0，可以避免获取InnoDB的表级锁。如果没有获取到InnoDB的表级锁，在其他事务锁定表中的一些数据的同时LOCK TABLES也能成功。
  
  在MySQL8.0中，设置innodb_table_locks=0对于LOCK TABLES ... WRITE这样明确指定锁表的操作无效，它只对通过LOCK TABLES ... WRITE隐式锁表的读和写（比如通过触发器），以及LOCK TABLES ... READ有效。

- InnoDB中所有的锁在事务提交或者终止都会释放，因此在 autocommit=1模式下执行LOCK TABLES没有意义，因为获取到的InnoDB表锁会直接释放掉。
- 在事务执行中不能再锁定其他表，因为LOCK TABLES会执行一个COMMIT 和UNLOCK TABLES操作。