---
layout: post
title: InnoDB架构（二）
categories: [MySQL]
description: InnoDB架构
keywords: MySQL, InnoDB
---

## InnoDB硬盘存储数据结构

### 双写缓冲区

双写缓冲区是InnoDB将缓冲池的的数据页刷新写入到InnoDB数据文件之前，写入的数据的地方。如果MySQL进程在写入数据页中途突然宕机，InnoDB可以从双写缓冲区找到数据页的备份来恢复数据。

虽然会写入两次，双写缓冲区并不需要两倍的I/O操作的性能消耗。数据是通过一个大的有序的数据块的方式写入双写缓冲区的，并且只进行一次fsync()系统调用（除非将`innodb_flush_method`设置为O_DIRECT_NO_FSYNC）。

### Redo日志

redo日志是基于硬盘的数据结构，用来恢复未完成事务的数据。redo日志会记录修改数据的操作，执行修改操作之后修改的内容还有写入到数据文件之前，MySQL服务器宕机，在MySQL服务器重启之后MySQL初始化时会自动执行重放操作恢复数据，在此之前MySQL不会接受外部的连接请求。

redo日志是存放在物理磁盘的redo日志文件中的。修改的数据会被记录到redo日志中，这些日志会用来恢复数据。当执行修改操作后，redo日志的记录会追加到redo日志文件中，随时间的推进redo日志文件中旧的数据会被截断。

#### 配置Redo日志的大小（MySQL8.0.30及以上版本）

从MySQL 8.0.30开始，`innodb_redo_log_capacity`系统变量控制redo日志文件占用磁盘空间大小。你可以在配置文件配置的这个变量，或者在运行时通过SET GLOBAL修改。例如，通过下面的语句可以将redo日志大小设置为8GB:
```
SET GLOBAL innodb_redo_log_capacity = 8589934592;
```
如果是在运行是修改，配置是直接生效的。如果redo日志文件占用的空间小于配置值，缓冲池中脏页刷盘到表空间的数据文件的频率会低一些，最终会增大redo日志的占用的磁盘空间。相关如果redo日志文件占用的空间大于配置值，缓存池中脏页刷盘频率就会高一些，最终会减少redo日志文件占用的空间。

`innodb_redo_log_capacity`变量取代了`innodb_log_files_in_group`和`innodb_log_file_size`变量，这两个变量已经过时。当定义了`innodb_redo_log_capacity`变量，`innodb_log_files_in_group`和`innodb_log_file_size`变量就会被忽略。否则，会通过`innodb_log_files_in_group`和`innodb_log_file_size`变量计算获得`innodb_redo_log_capacity`（innodb_log_files_in_group * innodb_log_file_size = innodb_redo_log_capacity）。如果这些变量都没有设置，redo日志的大小会设置为`innodb_redo_log_capacity`的默认值，默认值为104857600字节（100MB）。redo日志的最大容量是128GB。

Redo日志文件默认存在数据目录的#innodb_redo文件夹中，可以通过` innodb_log_group_home_dir`指定存放目录。redo日志文件分为两种：普通redo日志文件和备用redo日志文件。普通redo日志文件是正在使用的，备用redo日志文件是还没有使用的。InnoDB会维护32个redo日志文件，每个文件的大小为1/32 * innodb_redo_log_capacity。但是，在修改`innodb_redo_log_capacity`配置之后redo日志文件大小可能会不同。

Redo日志文件命名方式为#ib_redoN，N是redo日志文件的编号。备用redo日志文件以_tmp为后缀。下面例子展示#innodb_redo目录中的redo日志文件，有21个使用的redo日志文件，11个备用的redo日志文件，编号是有序的。
```
'#ib_redo582'  '#ib_redo590'  '#ib_redo598'      '#ib_redo606_tmp'
'#ib_redo583'  '#ib_redo591'  '#ib_redo599'      '#ib_redo607_tmp'
'#ib_redo584'  '#ib_redo592'  '#ib_redo600'      '#ib_redo608_tmp'
'#ib_redo585'  '#ib_redo593'  '#ib_redo601'      '#ib_redo609_tmp'
'#ib_redo586'  '#ib_redo594'  '#ib_redo602'      '#ib_redo610_tmp'
'#ib_redo587'  '#ib_redo595'  '#ib_redo603_tmp'  '#ib_redo611_tmp'
'#ib_redo588'  '#ib_redo596'  '#ib_redo604_tmp'  '#ib_redo612_tmp'
'#ib_redo589'  '#ib_redo597'  '#ib_redo605_tmp'  '#ib_redo613_tmp'
```
每个普通的redo日志文件关联一个范围的LSN（log sequence number）；例如，下面的查询展示了前面一个例子中的在使用的redo日志文件中的START_LSN和END_LSN：
```
mysql> SELECT FILE_NAME, START_LSN, END_LSN FROM performance_schema.innodb_redo_log_files;
+----------------------------+--------------+--------------+
| FILE_NAME                  | START_LSN    | END_LSN      |
+----------------------------+--------------+--------------+
| ./#innodb_redo/#ib_redo582 | 117654982144 | 117658256896 |
| ./#innodb_redo/#ib_redo583 | 117658256896 | 117661531648 |
| ./#innodb_redo/#ib_redo584 | 117661531648 | 117664806400 |
| ./#innodb_redo/#ib_redo585 | 117664806400 | 117668081152 |
| ./#innodb_redo/#ib_redo586 | 117668081152 | 117671355904 |
| ./#innodb_redo/#ib_redo587 | 117671355904 | 117674630656 |
| ./#innodb_redo/#ib_redo588 | 117674630656 | 117677905408 |
| ./#innodb_redo/#ib_redo589 | 117677905408 | 117681180160 |
| ./#innodb_redo/#ib_redo590 | 117681180160 | 117684454912 |
| ./#innodb_redo/#ib_redo591 | 117684454912 | 117687729664 |
| ./#innodb_redo/#ib_redo592 | 117687729664 | 117691004416 |
| ./#innodb_redo/#ib_redo593 | 117691004416 | 117694279168 |
| ./#innodb_redo/#ib_redo594 | 117694279168 | 117697553920 |
| ./#innodb_redo/#ib_redo595 | 117697553920 | 117700828672 |
| ./#innodb_redo/#ib_redo596 | 117700828672 | 117704103424 |
| ./#innodb_redo/#ib_redo597 | 117704103424 | 117707378176 |
| ./#innodb_redo/#ib_redo598 | 117707378176 | 117710652928 |
| ./#innodb_redo/#ib_redo599 | 117710652928 | 117713927680 |
| ./#innodb_redo/#ib_redo600 | 117713927680 | 117717202432 |
| ./#innodb_redo/#ib_redo601 | 117717202432 | 117720477184 |
| ./#innodb_redo/#ib_redo602 | 117720477184 | 117723751936 |
+----------------------------+--------------+--------------+
```
当生成一个checkpoint，InnoDB会将checkpoint的LSN存储在包含这个LSN的文件的头部。在恢复数据的时候，会检查所有的redo日志文件，从最近的checkpoint LSN开始恢复。

可以通过几个变量来查看redo日志大小是否允许调整和当前redo日志大小；例如，你可以`Innodb_redo_log_resize_status`查看调整大小操作的状态：
```
mysql> SHOW STATUS LIKE 'Innodb_redo_log_resize_status';
+-------------------------------+-------+
| Variable_name                 | Value |
+-------------------------------+-------+
| Innodb_redo_log_resize_status | OK    |
+-------------------------------+-------+
``` 
通过`Innodb_redo_log_capacity_resized`状态变量查看当前redo日志的大小限制：
```
mysql> SHOW STATUS LIKE 'Innodb_redo_log_capacity_resized';
 +----------------------------------+-----------+
| Variable_name                    | Value     |
+----------------------------------+-----------+
| Innodb_redo_log_capacity_resized | 104857600 |
+----------------------------------+-----------+
```
另外还有其他的一些变量：
- Innodb_redo_log_checkpoint_lsn

  redo日志的checkpoint LSN，这个变量是MySQL 8.0.30新增的。

- Innodb_redo_log_current_lsn

  当前LSN，代表redo日志缓冲区最近的写入位置。InnoDB会先把数据写入MySQL进程中的redo日志缓冲区，然后再请求操作系统把数据写入redo日志文件。这个变量是MySQL 8.0.30新增的。

- Innodb_redo_log_flushed_to_disk_lsn

  刷盘LSN。InnoDB首先将数据写入到redo日志，然后再请求操作系统将数据刷新到磁盘。刷盘LSN代表redo日志最近一次刷盘的位置，这样InnoDB知道从哪里开始执行刷盘操作。这个变量是MySQL 8.0.30新增的。

- Innodb_redo_log_logical_size
  
  redo日志所包含数据的大小，单位为字节，表示正在使用的redo日志数据的LSN值的范围，范围为redo日志消费需要的最旧的数据库到最新的数据库。这个变量是MySQL 8.0.30新增的。

- Innodb_redo_log_physical_size
  
  redo日志文件所占用的磁盘大小空间（不包含备用redo日志文件），单位为字节。这个变量是MySQL 8.0.30新增的。

- Innodb_redo_log_read_only

  redo日志是否是只读。这个变量是MySQL 8.0.30新增的。

- Innodb_redo_log_resize_status

  表示调整redo日志大小操作的状态。状态值包括：

  - OK：没有正在执行或者出现问题的调整redo日志大小的操作。
  - Resizing down：有一个调整redo日志大小的操作正在执行。
  
  调整redo日志大小的操作是瞬时的，所以没有正在执行的状态。这个变量是MySQL 8.0.30新增的。

- Innodb_redo_log_uuid
  
  redo日志的UUID。这个变量是MySQL 8.0.30新增的。

可以通过innodb_redo_log_files表来查询redo日志文件的信息：
```
SELECT FILE_ID, START_LSN, END_LSN, SIZE_IN_BYTES, IS_FULL, CONSUMER_LEVEL 
FROM performance_schema.innodb_redo_log_files;
```

#### 配置redo日志的大小（MySQL8.0.30以前的版本）

在MySQL8.0.30以前，InnoDB默认会在数据目录创建两个redo日志文件，命名为ib_logfile0和ib_logfile1，并且通过循环的方式写入文件。

可以通过修改redo日志文件数量和大小的方式来修改redo日志的大小：

1. 停止MySQL服务，并确保停止的时候没有发生错误。
2. 编辑my.cnf文件，通过`innodb_log_file_size`配置redo日志文件的大小，通过`innodb_log_files_in_group`修改redo日志文件的数量。
3. 启动MySQL服务。

如果InnoDB检测到`innodb_log_file_size`与redo日志文件的小大不同，会写入一个日志检查点，然后删除旧的日志文件，创建新的日志文件。

#### 自动redo日志大小配置

当`innodb_dedicated_server`开启的时候，InnoDB会自动配置InnoDB参数，包括redo日志的大小。自动配置一般是用在独立部署的MySQL服务器，服务器的所有资源只有MySQL使用。

### Undo日志

undo日志是关联一系列关联了读写事务的undo日志记录。一个undo日志包含了如何回滚到最近更新的事务的聚簇索引记录数据的所有信息。如果另外一个事务的一致性读需要看到原始数据，会从undo日志记录中获取。undo日志存在于undo日志段中，undo日志段包含在回回滚段中，回滚段存在于undo表空间和全局临时表空间中。

在全局临时表空间中的undo日志用于修改用户定义的临时表数据的事务。这些undo日志不是用来崩溃恢复的，主要是用来回滚数据。

每个undo表空间和全局临时表空间支持最大128个回滚段，` innodb_rollback_segments`定义回滚段的数量。

一个回滚段支持的事务的数量取决于回滚段中undo插槽的数量和每个事务需要的undo日志的数量。不同大小的InnoDB数据页的回滚段的undo插槽的数量是不同的。

|InnoDB数据页大小|回滚段的undo插槽的数量(InnoDB数据页大小/16)|
|:-|:-|
|4096 (4KB)|256|
|8192 (8KB)|512|
|16384 (16KB)|1024|
|32768 (32KB)|2048|
|65536 (64KB)|4096|

一个事务最多分配4个undo日志，每个undo日志对应下面操作类型：
1. 用户定义的表的INSERT操作
2. 用户定义的表的UPDATE和DELETE操作
3. 用户定义的临时表的INSERT操作
4. 用户定义的临时表的UPDATE和DELETE操作

undo日志是按照需要分配的。例如，一个在普通表和临时表上执行了INSERT,UPDATE和DELETE操作的事务，需要4个undo日志。一个在普通表上只执行了INSERT操作的事务只需要一个undo日志。