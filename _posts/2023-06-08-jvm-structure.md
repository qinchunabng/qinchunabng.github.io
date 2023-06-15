---
layout: post
title: JVM整体结构
categories: [Java]
description: JVM整体结构
keywords: Java, JVM
---

## JVM整体结构

![JVM整体结构](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/java/JVM%E6%95%B4%E4%BD%93%E7%BB%93%E6%9E%84.png?raw=true)

上图为HotSpot JVM结构图，HotSpot是目前市面上高性能虚拟机的代表之一。它采用解释器与即时编译并存的架构。
HotSpot JVM结构从上向下分为：
- 类装载子系统。
  
  类装载子系统也叫类加载器，用来加载Class文件。
- 运行时数据区。
  
  运行时数据区在程序执行过程中使用，运行时数据区分为：
  - 程序计数器。

    Java虚拟机中同时可以运行多个线程，每个线程都拥有自己的程序计数器。在任何时刻，每个Java虚拟机线程只能执行一个方法中的代码。如果执行的方式不是本地方法，程序计数器包含了当前执行Java虚拟机指令的地址。如果当前执行的方法是本地方法，Java虚拟机的pc寄存器的值是undifined。
  - Java虚拟机栈。

    每个Java虚拟机线程都有一个私有的Java虚拟机栈，在线程创建的同时创建。Java虚拟机栈中存储的是栈帧。栈帧是用来存储数据和中间运算结果，以及方法的返回值，执行动态连接，分发异常。栈帧在方法被调用的时候被创建，在方法执行结束后被销毁。与Java虚拟机栈相关的异常有：
    - 如果一个线程需要Java虚拟机栈的大小超过限制，Java虚拟机会抛出StackOverflowException。
    - 如果Java虚拟机栈可以动态扩容，但是又没有足够的内存，或者在创建一个新的线程的虚拟机栈时没有足够的内存，Java虚拟机会抛出OutOfMemoryError。
  - 堆。

    在Java虚拟机中堆是所有的线程共享的，堆是一个运行时数据区，所有类的实例和数组分配内存都在堆上。堆是在JVM启动的时候创建的。堆中存储的对象内存空间是通过自动内存管理系统（即垃圾收集器）回收的，无需明确释放。与堆有关的异常：
    - 如果程序运行过程中需要堆内存超过限制，会抛出OutOfMemoryError。
  - 方法区。

    方式区是被所有的线程共享的。方法区类似一些其他语言中用来存储已编译代码的存储区域。它存储每个类的结构，比如运行时常量池，字段和方法的信息，以及方法和构造函数的代码，包括在类和实例初始化和结构初始用到的一些特殊的方法。方法区时在JVM启动的时候创建的。虽然方法区时堆的一个逻辑区域，但它不会被垃圾回收。与方法区有关的异常：
    - 如果方法区无法满足内存的分配请求，JVM会抛出OutOfMemoryError。
  - 运行时常量池。

    运行时常量池是运行时来存放class文件中常量池表的数据。它包含了几种常量，从在编译时就知道数字字面量到字段和方法在运行时必须解析的引用。运行时常量池的作用类似传统语言的符号表，虽然它包含的数据比符号表更广泛。运行时常量池是在JVM方法区上分配的，当类和结构创建的时候其对应常量池也会被创建。与运行时常量池相关的异常：
    - 当前创建一个类或接口，创建运行时常量池需要的内存超过方法区的大小，JVM会抛出OutOfMemoryError。
  - 本地方法栈。

    Java为了支持本地方法，可能会使用到叫做“C栈”的传统栈。本地方法栈也可以被JVM指令集解释器（比如用C语言实现的）使用。本地方法栈是在线程创建的时候创建的。与本地方法区相关的异常：
    - 如果线程需要的本地方法栈的超过大小限制，JVM会抛出StackOverflowError。
    - 如果本地方法栈可以动态扩展但是没有足够的内存，或者在线程创建的时候创建初始化本地方法栈没有足够的内存，JVM会抛出OutOfMemoryError。
  
- 执行引擎。
  
  执行引擎包含了解释器、即时编译器、垃圾回收器。字节码加载到虚拟机要执行需要通过解释器，但是对于一些热点代码，每次都去解释执行，效率会很慢。所以就有了即时编译器，即时编译器会提前将热点代码编译为机器码，提高执行效率。垃圾回收器实现垃圾的回收。总的来说执行引擎的作用就是将字节码转为可以执行的机器码。

### Java代码执行流程

![Java代码执行流程](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/java/Java%E4%BB%A3%E7%A0%81%E6%89%A7%E8%A1%8C%E6%B5%81%E7%A8%8B.png?raw=true)

Java代码经过编译器编译成一个或多个字节码文件，在编译过程中会经过词法分析、语法分析、语法/抽象语法树、语义分析、注解抽象语法树、字节码生成器生成字节码文件。在编译的过程中任何一个环节失败都会编译失败。字节码经过类加载器的加载，在经过字节码校验器校验，最后再翻译字节码解释执行，或者通过JIT编译器编译执行。解释执行/编译执行这块相当于JVM整体结构中的执行器。操作系统并不识别字节码指令，只能识别机器指令，执行器所作的工作就是就是把字节码翻译为机器中指令。正常情况字节码是通过逐行解释执行的，但是一些反复多次执行的热点代码，执行器会采用JIT编译器将字节码编译为机器指令，并将编译后的机器指令放到方法区缓存起来，下次指令直接调用。

那如果通过JIT编译之后执行速度会更快，为什么不把所有的字节通过JIT编译之后缓存起来？不这么做是因为如果都使用JIT遍历，在程序启动的时候编译时间会过长，导致卡顿，为了提高响应时间，所以采用折中的方案，解释执行和编译执行共存。