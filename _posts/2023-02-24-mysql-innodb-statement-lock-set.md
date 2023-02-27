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