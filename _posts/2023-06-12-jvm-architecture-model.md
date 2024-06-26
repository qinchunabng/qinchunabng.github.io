---
layout: post
title: JVM的架构模型和生命周期
categories: [Java]
description: JVM的架构模型和生命周期
keywords: Java, JVM
---

## JVM的架构模型

![JVM整体结构](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/java/JVM%E6%9E%B6%E6%9E%84-%E8%8B%B1.jpg?raw=true)

Java编译器输入的指令流基本上是一种基于栈的指令集架构，另一种指令架构则是基于基于寄存器的指令集架构。

这两种架构之间的区别：
- 基于栈式架构的特点
  - 设计和实现更简单，适用于资源受限的系统
  - 避开寄存器的分配难题：使用零地址指令方式分配
  - 指令流中的指令大部分是零地址指令，其执行过程依赖于操作栈。指令集更小，编译器更容易实现。
  - 不需要硬件支持，可移植性更好，更好的实现跨平台。
- 基于寄存器架构的实现特点
  - 典型的架构是x86的二进制指令集：比如传统的PC以及Android的Davlik虚拟机。
  - 指令集架构完全依赖硬件，可以执行差
  - 性能更优秀和执行更高效
  - 花费更好的指令完成一项操作
  - 在大多数情况下，基于寄存器的指令集往往都以一地址指令、二地址指令和三地址指令为主，而基于栈式架构的指令集却是以零地址指令为主

总结：由于跨平台性的设计，Java的指令都是根据栈来设计的。不同平台CPU架构不同，所以不能设计为基于寄存器。优点是跨平台，指令集小，编译器容易实现，缺点是性能下降，实现同样的功能需要更多的指令。

## JVM的生命周期

### 虚拟机的启动

Java虚拟机的启动是通过引导类加载器（bootstrap class loader）创建一个初始类（initial class）来完成，这个类是由虚拟机的具体实现指定的。

### 虚拟机的执行

- 一个运行中的Java虚拟机有着一个清晰的任务：执行Java程序。
- 程序开始执行时它才执行，程序结束时它就停止。
- 执行一个Java程序的时候，真正执行的是一个Java虚拟机进程。

### 虚拟机的退出

有如下几种情况：

- 程序正常执行结束
- 程序在执行过程中遇到了异常或错误而终止
- 由于操作系统出现错误而导致Java虚拟机进制终止
- 某线程调用Runtime类或System类的exit方法，或Runtime类的halt方法，并且Java安全管理器也允许这次exit或halt操作。
- 除此之外，JNI(Java Native Interface)规范描述了用JNI Invocation API来加载或卸载Java虚拟机时，Java虚拟机的退出情况。

