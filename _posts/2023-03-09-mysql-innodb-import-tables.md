---
layout: post
title: 导入导出InnoDB表
categories: [MySQL]
description: 导入导出InnoDB表
keywords: MySQL, InnoDB
---

## 导入InnoDB表

InnoDB可以导入有独立表空间的普通表，分区表或者分区表的某个分区。导入表有以下场景：
  
- 在非生产的MySQL实例上生成报告，避免给生产服务造成压力。
- 复制数据到一个新的副本服务。
- 从备份表空间文件恢复数据。
- 作为一种比从dump文件导入数据更快方式导入数据，因为从dump文件导入需要重新插入数据并且重建索引。
- 将数据转移到一个更适合存储的服务器，例如，你可能会将一个使用频率很高的表移动到SSD服务器，或者移动一个大表到一个高容量的HDD设备。
  
### 先决条件

- `innodb_file_per_table`必须开启，默认开启。
- 导入的MySQL实例的页大小必须与原表的表空间的页大小一致。InnoDB页大小是通过`innodb_page_size`定义的，是在初始化MySQL的时候配置的。
- 如果表有外键，在执行`DISCARD TABLESPACE`前要先关闭`foreign_key_checks`，同时导出表相关的所有外键，因为`ALTER TABLE ... IMPORT TABLESPACE`不强制导入外键约束。执行这个操作的时候，需要停止更新表，提交所有的事务，获取表的共享锁，然后执行导出操作。
- 当如导入数据时，导出和导入的MySQL实例都必须是稳定的状态，且数据库版本要保持一致。否则，必须在要导入的MySQL实例上先创建表。
- 如果表是创建时通过`DATA DIRECTORY`指定外部数据目录创建的，到导入的表也应该有同样的`DATA DIRECTORY`条件。否则，会出现schema不匹配的异常。通过`SHOW CREATE TABLE t`语句查看表是否指定`DATA DIRECTORY`条件。
- 如果`ROW_FORMAT`没有明确指定，将会使用`ROW_FORMAT=DEFAULT`，MySQL的导入导出实例的`innodb_default_row_format`参数应该保持一致。否则，导入时会出现schema不匹配的错误。通过`SHOW VARIABLES`查看`innodb_default_row_format`参数。

### 导入表

这个例子演示如果导入一个使用独立表空间的不分区的普通表。

1. 在导入的目标实例中，创建一个跟导出表一模一样的表（可以通过`SHOW CREATE TABLE`查看见表语句）。
   ```
   mysql> USE test;
   mysql> CREATE TABLE t1 (c1 INT) ENGINE=INNODB;
   ```
2. 在目标实例中，删除刚刚创建表的表空间。
   ```
   mysql> ALTER TABLE t1 DISCARD TABLESPACE;
   ```
3. 在源实例中，在要导出的表上执行`FLUSH TABLES ... FOR EXPORT`，执行之后只允许查询这个表。
   ```
   mysql> USE test;
   mysql> FLUSH TABLES t1 FOR EXPORT;
   ```
   `FLUSH TABLES ... FOR EXPORT`保证表中的所有改变都刷新到硬盘中，这样表的二进制副本才能被创建。当执行`FLUSH TABLES ... FOR EXPORT`时，InnoDB会在表的schema目录生成`.cfg`的元数据文件。`.cfg`包含导入数据时验证schema所需要的元数据。
   
   > **注意**
   > 
   > 在执行`FLUSH TABLES ... FOR EXPORT`时，数据库连接不能断开，否则连接一断开锁就会被释放`.cfg`文件就会被移除。
4. 复制`.ibd`文件和`.cfg`的元数据文件到目标实例中。例如：
   ```
   $> scp /path/to/datadir/test/t1.{ibd,cfg} destination-server:/path/to/datadir/test
   ```
   `.ibd`文件和`.cfg`需要在锁释放前拷贝过去。
   > **注意**
   > 
   > 如果你是从一个加密的表空间导入数据的，InnoDB会生成一个额外的`.cfp`文件。`.cfp`文件需要和`.cfg`一起拷贝到目标实例。`.cfp`文件中包含了一个传输密钥和一个表空间加密密钥。导入时，InnoDB使用传输密钥解密表空间密钥。

5. 在源实例中，使用`UNLOCK TABLES`来释放`FLUSH TABLES ... FOR EXPORT`加的锁：
   ```
   mysql> USE test;
   mysql> UNLOCK TABLES;
   ```
   `UNLOCK TABLES`操作会移除`.cfg`文件。

6. 在目标实例，导入表空间：
   ```
   mysql> USE test;
   mysql> ALTER TABLE t1 IMPORT TABLESPACE;
   ```

