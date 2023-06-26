---
layout: post
title: JVM的发展历程
categories: [Java]
description: JVM的发展历程
keywords: Java, JVM
---

## JVM的发展历程

### SUN公司的HotSpot VM

HotSpot历史

- 最初由一家名为“Longview Technologies”的小公司设计。
- 1997年，此公司被Sun收购；2009年，Sun公司被甲骨文收购。
- JDK1.3时，HotSpot VM成为默认虚拟机。

当前HotSpot占有绝对的市场地位，称霸武林。

- 不管是现在仍在广泛使用的JDK6，还是使用比例较多的JDK8中，默认的虚拟机都是HotSpot。
- Sun/Oracle JDK和OpenJDK的默认虚拟机。

从服务器、桌面到移动端、嵌入式都有应用。

名称中的HotSpot指的就是它的热点代码探测技术。

- 通过计数器找到最具编译价值代码，触发即时编译或栈上替换。
- 通过编译器与解释器协同工作，在最优化的程序响应时间与最佳执行性能中取得平衡。

### BEA的JRockit

专注于服务器端应用

- 它可以不太关注程序启动速度，因此JRockit内部不包含解析器实现，全部代码都靠即时编译器编译后执行。

大量的行业基准测试显示，JRockit JVM是世界上最快的JVM。

- 使用JRockit产品，客户已经体验到了显著的性能提高（一些超过了70%）和硬件成本的减少（达50%）。

优势：全面的Java运行时解决方案组合

- JRockit面向延迟敏感型应用的解决方案JRockit Real Time提供以毫秒或微秒级的JVM响应时间，适合财务、军事指挥、电信网络的需要。
- MissionControl服务套件，它是一组以极低的开销来监控、管理和分析生产环境中的应用程序的工具。

2008年，BEA被Oracle收购。

Oracle表达了整合两大优秀虚拟机的工作，大致在JDK8中完成。整合的方式是在HotSpot的基础上完成，移植JRockit的优秀特性。

### IBM的J9

- 全称：IBM Technology for Java Virtual Machine，简称IT4J，内部代号：J9
- 市场定位与HotSpot接近，服务器端、桌面应用、嵌入式等多用途VM
- 广泛应用于IBM的各种Java产品。
- 目前，有影响力的三大商用虚拟机之一，也是号称世界上最快的的Java虚拟机。
- 2017年左右，IBM发布了开源J9 VM，命名为OpenJ9，交给Eclipse基金会管理，也成为Ecplise OpenJ9。

### Dalvik VM

- Dalvik VM是谷歌开发的，应用于Anroid系统，并在Android2.2提供了JIT，发展迅猛。
- Dalvik VM只能称作虚拟机，而不能称为“Java虚拟机”，它没有遵循Java虚拟机规范。
- 不能执行Java的Class文件。
- 基于寄存器架构，不是jvm的栈架构。
- 执行的是编译后的dex（Dalvik Executable）文件，执行效率比较高。
  - 它执行的dex（Dalvik Executable）文件可以通过Class文件转而来，使用Java语法编写应用程序，可以直接使用大部分的Java API等。

- Android 5.0使用支持提前编译（Ahead of Time Compilation，AOT）的AVT VM替换Dalvik VM。
