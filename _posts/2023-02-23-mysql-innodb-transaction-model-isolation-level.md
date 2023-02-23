---
layout: post
title: MySQL InnoDB事务模型之事务隔离级别
categories: [MySQL]
description: InnoDB的事务隔离
keywords: 数据库, MySQL, InnoDB, 事务隔离级别
---

事务隔离级别是数据库事务的最重要的特性之一，是事务四大特性ACID中的I。通过调整事务隔离级别可以对事务的性能、可靠性、一致性之间的平衡进行调整。

InnoDB提供了四个事务隔离级别：READ_UNCOMMITED,READ_COMMITTED,REPEATABLE_READ和SERIALIZABLE，默认隔离级别是REPEATBLE_READ。可以通过`SET TRANSACTION`调整单个数据库会话的事务隔离级别。设置MySQL事务隔离级别，还可以通过`--transaction-isolation`命令行参数设置，设置将对所有数据库连接生效。

InnoDB的事务隔离级别使用不同的锁策略。如果对数据的一致性要求较高，使用默认的REPEATABLE_READ。如果对数据的一致性要求没有那么高，或者不需要做到可重复读，可以使用READ_COMMITTED，可以降低锁的开销。SERIALIZABLE具有比REPEATBLE_READ有更强的一致性保证，但是性能最差，一般用在一些特殊的场景，比如XA事务或者排查并发和死锁问题。

下面详细了解一下四种不同事务隔离级别，从上到下按照使用场景的多少排列，越往下使用场景越少。

- REPEATBLE_READ

  这是InnoDB默认事务隔离级别。当使用REPEATBLE_READ隔离级别时，同一个事务中普通的SELECT查询（不加锁），同样查询语句多次查询到数据是一样的，读取 的都是第一次查询的时数据快照。
  
  对于加锁的读（SELECT...FOR UPDATE或SELECT...FOR SHARE）,UPDATE,和DELETE语句，怎么加锁取决于语句是否使用唯一索引或者范围查找作为查询条件。

  - 如果使用唯一索引作为查询条件，InnoDB只会锁定查找到索引记录，不会锁定间隙。

  - 对于其他情况，InnoDB会使用间隙锁或者next-key锁锁定扫描到的索引范围，防止其他的会话往数据间隙中插入数据。

- READ_COMMITTED
  
  使用该隔离级别，同一个事务中每次读取都会回去最新数据快照。
  
  对于加锁的读（SELECT...FOR UPDATE或SELECT...FOR SHARE）,UPDATE,和DELETE语句，只会锁对应的索引记录，不会锁数据间的间隙，因此可以随便插入数据到间隙中。间隙锁只会在外键检查或者重复值检查的时候才会用到。因为禁用的间隙锁，幻读的问题就可能会发生，因为其他会话可以插入数据到间隙中。

  只有基于行的二进制日志支持READ COMMITTED事务隔离级别，如果设置binlog_format=MIXED，MySQL将使用基于行的日志记录。

  使用READ COMMITTED还有一下影响：

  - 对于UPDATE和DELETE语句，InnoDB只会索引要更新和删除的行。MySQL在评WHERE查询条件后，将会释放不匹配的记录锁，这可以大大降低思索发生的概率。

  - 对于UPDATE语句，如果数据已经被锁住，InnoDB就会执行半一致性读，返回最新提交数据版本，这样MySQL可以确定哪些数据匹配到WHERE条件。如果有数据被匹配到，MySQL将会再次读取数据行，然后再锁住或者等待锁。

  考虑下面的例子，有这样一张表：
  ```
  CREATE TABLE t (a INT NOT NULL, b INT) ENGINE = InnoDB;
  INSERT INTO t VALUES (1,2),(2,3),(3,2),(4,3),(5,2);  
  COMMIT;
  ```
  在这个例子中，表没有索引，所以查询和索引扫描将使用隐藏的聚集索引来获取记录锁。

  加入有一个会话使用下面语句执行UPDATE操作：
  ```
  # Session A
  START TRANSACTION;
  UPDATE t SET b = 5 WHERE b = 3;
  ```
  假设有另外一个会话在之后执行了下面UPDATE语句：
  ```
  # Session B
  UPDATE t SET b = 4 WHERE b = 2;
  ```
  由于InnoDB执行UPDATE操作时，首先要获取所有行的独占锁，然后再判断是否需要修改。如果InnoDB不修改该行，会释放锁。否则InnoDB会保留锁直到事务结束。