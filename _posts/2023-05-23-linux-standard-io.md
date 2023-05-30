---
layout: post
title: 标准I/O
categories: [Linux]
description: 标准I/O
keywords: C, Linux编程
---

## 标准I/O

I/O即输入和输出，I/O是一切实现的基础，如果没有I/O数据就无法保存。I/O分为两种：stdio标准IO和sysio系统调用IO（文件IO）。在两种都可以使用的时候，优先使用标准IO。

### 系统调用IO和标准IO的区别

我们的程序一般都运行在用户态，如果要与内核态进行对话时，需要通过提供系统调用函数。如果系统不一样，提供的系统函数就不一样，就会给用户造成困扰。这个时候标准就出现了，来解决不同系统的系统调用不一样的问题。标准IO提供标准，各个平台各自去实现。所以在标准IO和系统调用IO都可用的时候，优先使用标准，移植性好。

### 标准IO

标准IO（stdio）常见函数：
```
fopen();
fclose();
fgetc();
fgets();
fputs();
fread();
fwrite();

printf();
scanf();

fseek();
ftell();
rewind();

fflush();
```

了解一个函数的使用方法，可以使用man手册查询。比如要查看fopen使用方法，可以使用如下命令：
```
man fopen
```
man手册的第1章是基本的命令，第2章是系统调用，第3章是标准库函数，第7章讲机制，可以通过如下命令指定第几章：
```
man 3 fopen
```
以上命令明确指定从第3章查询fopen函数。