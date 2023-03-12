---
layout: post
title: MySQL索引下推
categories: [MySQL]
description: MySQL索引下推
keywords: 数据库, MySQL, 索引下推
---

索引下推是MySQL对索引查询的优化。没有索引下推,存储引擎会遍历索引从表中定位数据后，返回给MySQL服务层，MySQL服务层再根据WHERR中条件判断数据是否匹配。如果开启索引下推，并且WHERE部分查询条件只能用到索引查询，MySQL服务层将会把WHERE条件的判断下推到存储引擎完成。
存储引擎通过索引数据判断是否满足WHERE中的条件，只有满足条件才从表中去读取行数据。索引下推可以减少存储引擎访问表的次数以及减少MySQL服务层访问存储引擎的次数。

索引下推的适用性取决一下条件：
- 当需要访问完整的数据行数据时，并且查询类型是range,ref,eq_ref和ref_or_null时，索引下推会被使用。
  
- InnoDB和MyISAM都可以使用索引下推，包括分区表。
  
- 对于InnoDB表，索引下推只有二级索引会用到。索引下推的目的是检查二级索引读取整行数据进行的回表查询，以此减少IO操作。对于聚簇索引，由于聚簇索引所有整行数据，在查询索引的时候已经加载到InnoDB缓冲区，在这种情况下索引下推不会减少IO。
  
- 引用子查询的条件不会用到索引下推。
  
- 存储方法不会用到索引下推，因为存储引擎无法调用存储方法。
  
- 触发器不会用到索引下推。
  
- 包含对系统变量引用的派生表不能使用索引下推。

理解索引下推是如何工作的，首先我们要了解没有索引下推时索引扫描是怎么执行的：

- 获取下一行的索引记录，通过索引记录定位读取整行数据。
  
- 根据数据判断是否满足WHERE条件，以此判断是否在结果集中返回这行数据。

如果使用索引下推，索引扫描将会按照下面的方式执行：

- 获取下一行的索引记录（不包含整行数据）。
  
- 使用索引列判断是否满足WHERE条件，如果不满足继续扫描下一行的索引。
  
- 如果满足条件，则通过索引定位获取整行数据。
  
- 检查是否满足剩下WHERE的条件，根据检查结果决定是否在结果集中返回。

可以通过EXPLAIN查看是否使用索引下推，如果Extra列中显示Using index condtion表示使用到索引下推。不是显示Using index,因为必须读取整行数据。

下面通过一个例子演示索引下推。假设有一张表people，包含INDEX (zipcode, lastname, firstname)这样的复合索引。如果我们知道某个人的zipcode，但是不太清楚他的lastname和firstname，那么我们SQL可能会这么写：
```
SELECT * FROM people
WHERE zipcode='95054'
AND lastname LIKE '%etrunia%'
AND address LIKE '%Main Street%';
```
MySQL会使用扫描索引获取`zipcode='95054'`的数据，但是`lastname LIKE '%etrunia%'`无法被使用到，所以如果没有索引下推，这个查询会获取全表中所有`zipcode='95054'`的数据。

开启索引下推，MySQL在读取整行数据前会检查`lastname LIKE '%etrunia%'`的条件，这会避免扫描匹配`zipcode='95054'`但是不匹配`lastname LIKE '%etrunia%'`的条件的整行数据。

索引下推是默认开启的，可以通过optimizer_switch系统变量来设置：
```
SET optimizer_switch = 'index_condition_pushdown=off';
SET optimizer_switch = 'index_condition_pushdown=on';
```