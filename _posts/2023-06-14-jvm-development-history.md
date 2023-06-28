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

### Azul VM

- 前面提到的三大“高性能Java虚拟机”使用在通用硬件平台上。
- 这里Azul VM和EBA Liquid VM是与特定硬件平台绑定、软硬件配合的专有虚拟机。
  - 高性能Java虚拟机中的战斗机。
- Azul VM是Azul Systems公司在HotSpot基础上进行大量改进，运行于Azul Systems公司的专有硬件Vega系统上的Java虚拟机。
- 每个Azul VM实例都可以管理至少数十个CPU和数百GB内存的硬件资源，并提供在巨大内存范围内实现可控的GC时间的垃圾收集器、专有硬件优化的线程调度等优秀特性。
- 2010年，Azul Systems公司开始硬从硬件转向软件，发布了自己的Zing JVM，可以在通用的x86平台上提供接近Vega系统的特性。

### Liquid VM

- 高性能Java虚拟机中的战斗机。
- EBA公司开发的，直接运行在自家Hypervisor系统上。
- Liquid VM即是现在的JRockit VE（Virtual Edition），Liquid VM不需要操作系统的支持，或者说它本身实现了一个专用操作系统的必要功能，如线程调度、文件系统、网络支持等。
- 随着JRockit虚拟机终止开发，Liquid VM项目也停止了。
  
### Apache Harmony

- Apache也曾今推出过与JDK1.5和JDK1.6兼容的Java运行平台Apache Harmony。
- 它是IBM和Intel联合开发的开源JVM，受到同样开源的OpenJDK的压制，Sun坚决不让Harmony获得JCP认证，最终于2011退役，IBM转而参与OpenJDK。
- 虽然目前并没有Apache Harmony被大规模商用的案例，但是它的Java类库代码被吸纳进了Android SDK。

### Microsoft JVM

- 微软为了在IE3浏览器中支持Java Applets，开发了Microsoft JVM。
- 只能在Windows平台下运行，但却是当时在Windows下性能最好的Java VM。
- 1997年，Sun以侵犯商标、不正当竞争罪名指控微软成功，赔了Sun很多钱。微软在WindowsXP SP3中抹掉了其VM。现在Windows上安装的jdk都是HotSpot。
  
### TaobaoVM

- 有AliVM团队发布。阿里，国内使用Java最强的公司，覆盖云计算、金融、物流、电商等众多领域，需要解决高并发、高可用、分布式的复合问题。有大量的开源产品。
- 基于OpenJDK开发了自己的定制版AlibabaJDK，简称AJDK，是整个阿里Java体系的基石。
- 基于OpenJDK HotSpot VM发布的国内第一个优化、深度定制且开源的高性能服务器版的Java虚拟机。
  - 创新的GCIH(GC invisible heap)技术实现了off-heap，即将生命周期较长的Java对象从heap中移到heap之外，并且GC不能管理GCHI内部的Java对象，以此达到降低GC的回收频率和提升GC的回收效率的目的。
  - GCIH中的对象还能够在多个Java虚拟机进程中实现共享。
  - 使用crc32指令实现JVM intrinsic降低JNI的调用开销。
  - PMU hardware的Java profiling tool和诊断协助功能。
  - 针对大数据场景的ZenGC。
- taobao vm应用在阿里产品上性能高，硬件严重依赖intel的cpu，损失了兼容性，但提高了性能。
  - 目前已经在淘宝、天猫上线，把Oracle官方的JVM版本全部替换了。
  

### Dalvik VM

- Dalvik VM是谷歌开发的，应用于Anroid系统，并在Android2.2提供了JIT，发展迅猛。
- Dalvik VM只能称作虚拟机，而不能称为“Java虚拟机”，它没有遵循Java虚拟机规范。
- 不能执行Java的Class文件。
- 基于寄存器架构，不是jvm的栈架构。
- 执行的是编译后的dex（Dalvik Executable）文件，执行效率比较高。
  - 它执行的dex（Dalvik Executable）文件可以通过Class文件转而来，使用Java语法编写应用程序，可以直接使用大部分的Java API等。

- Android 5.0使用支持提前编译（Ahead of Time Compilation，AOT）的AVT VM替换Dalvik VM。


###  Graal VM

- 2018年4月，Oracle Labs公布了Graal VM，号称"Run Programs Faster Anywhere"，野心勃勃。与1995年的Java的"write once,run anywhere"遥相呼应。
- Graal VM在HotSpot VM基础上增强而成的跨语言全栈虚拟机，可以作为“任何语言”的运行平台使用。语言包括：Java、Scala、Groovy、Kotlin;C、C++、JavaScript、Ruby、Python、R等。
- 支持不同语言中混用对方的接口和对象，支持这些语言使用已经编写好的本地库文件。
- 工作原理是将这些原因的源代码或源代码编译后的中间格式，通过解释器转换为能被Graal VM接受的中间表示。Graal VM提供了Truffle工具快速构建面向一种新语言的解释器。在运行时还能够进行即时编译优化，获得比原生编译器更优秀的执行效率。
- 如果说HotSpot有一天真的被取代，Graal VM希望最大。但是Java的软件生态没有丝毫变化。