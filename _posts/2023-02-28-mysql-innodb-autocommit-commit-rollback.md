---
layout: post
title: MySQL InnoDB中的autocommit,commit,rollback
categories: [MySQL]
description: MySQL InnoDB中的autocommit,commit,rollback
keywords: 数据库, MySQL, InnoDB, autocommit, commit, rollback
---

在InnoDB中，所有的数据操作就是在事务中执行的。如果autocommit开启，每个SQL语句就是一个事务，默认情况下，MySQL所有的会话autocommit都是默认开启的，所以每个SQL语句执行没有错误MySQL会自动执行提交操作。如果SQL执行报错，根据错误类型判断是否执行提交或者回滚。

在autocommit开启的情况下，可以通过START TRANSACTION或者BEGIN开启事务，COMMIT和ROLLBACK提交回滚事务，使一个事务中可以有多条SQL语句执行。

如果autocommit模式关闭，每个执行的会话总是会有一个事务开启，COMMIT或者ROLLBACK结束事务，然后开启另一个事务。

如果autocommit关闭的情况下，一个会话没有明确的提交事务，会话结束时MySQL将会回滚这个事务。

有些SQL语句会隐式的提交事务，就如同执行语句之前执行COMMIT操作一样，比如DDL语句都会隐式的提交事务。

COMMIT操作以为着事务中所有的修改将会被持久化到磁盘上，所作的修改也会对其他事务可见，ROLLBACK操作是取消事务中所作的修改。COMMIT和ROLLBACK操作都会释放事务中加的锁。

默认情况下，MySQL的autocommit都是开启的，这时所有的操作都会自动提交，这和有些其他的数据库不一样。

要使一个事务中包含多条SQL执行语句，可以通过SET autocommit = 0 关闭自动提交，每个事务在结束时执行COMMIT或者ROLLBACK。或者不关闭autocommit，使用START TRANSACTION开启一个事务， COMMIT或 ROLLBACK来结束事务。下面的例子演示两种事务的开启方式：
```
mysql> CREATE TABLE customer (a INT, b CHAR (20), INDEX (a));
Query OK, 0 rows affected (0.00 sec)
mysql> -- Do a transaction with autocommit turned on.
mysql> START TRANSACTION;
Query OK, 0 rows affected (0.00 sec)
mysql> INSERT INTO customer VALUES (10, 'Heikki');
Query OK, 1 row affected (0.00 sec)
mysql> COMMIT;
Query OK, 0 rows affected (0.00 sec)
mysql> -- Do another transaction with autocommit turned off.
mysql> SET autocommit=0;
Query OK, 0 rows affected (0.00 sec)
mysql> INSERT INTO customer VALUES (15, 'John');
Query OK, 1 row affected (0.00 sec)
mysql> INSERT INTO customer VALUES (20, 'Paul');
Query OK, 1 row affected (0.00 sec)
mysql> DELETE FROM customer WHERE b = 'Heikki';
Query OK, 1 row affected (0.00 sec)
mysql> -- Now we undo those last 2 inserts and the delete.
mysql> ROLLBACK;
Query OK, 0 rows affected (0.00 sec)
mysql> SELECT * FROM customer;
+------+--------+
| a    | b      |
+------+--------+
|   10 | Heikki |
+------+--------+
1 row in set (0.00 sec)
mysql>
```

