---
layout: post
title: InnoDB行格式
categories: [MySQL]
description: InnoDB行格式
keywords: MySQL, InnoDB
---

### InnDB行格式

表的行格式决定了如何物理存储数据行，数据的物理存储方式会影响查询和DML操作的性能。一个磁盘页的存储的数据行越多，查询和索引查找越快，需要的缓冲池越小，更新操作的I/O越少。