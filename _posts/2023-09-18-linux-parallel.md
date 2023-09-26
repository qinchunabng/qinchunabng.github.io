---
layout: post
title: Linux-并发
categories: [Linux]
description: Linux-并发
keywords: Linux, C
---

## 并发

同步就是执行执行的时候什么时候会执行什么代码是清楚的。异步就是什么事件时候到来不知道，会产生什么样的结果也不知道。

异步事件的处理：查询法和通知法。举个例子：一个人在河边钓鱼，鱼有没有上钩就是一个异步事件，鱼什么时候上钩是不知道。如果想知道有没有鱼上钩，过一会用抄网捞一下，看看有没有鱼上钩，这种方式就属于查询法。另外一份方式就是用鱼漂，有鱼上钩鱼漂会动，这就是通知法。

什么时候用通知法什么时候用查询法？如果当前异步事件发生的比较稀疏，用通知法比较合适。如果发生的频率比较高，使用查询发。

### 信号

信号是软件中断。信号的响应依赖于中断。

#### 信号的概念

通过`kill -l`命令可以查看所有的信号：
```
 1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL       5) SIGTRAP
 6) SIGABRT      7) SIGBUS       8) SIGFPE       9) SIGKILL     10) SIGUSR1
11) SIGSEGV     12) SIGUSR2     13) SIGPIPE     14) SIGALRM     15) SIGTERM
16) SIGSTKFLT   17) SIGCHLD     18) SIGCONT     19) SIGSTOP     20) SIGTSTP
21) SIGTTIN     22) SIGTTOU     23) SIGURG      24) SIGXCPU     25) SIGXFSZ
26) SIGVTALRM   27) SIGPROF     28) SIGWINCH    29) SIGIO       30) SIGPWR
31) SIGSYS      34) SIGRTMIN    35) SIGRTMIN+1  36) SIGRTMIN+2  37) SIGRTMIN+3
38) SIGRTMIN+4  39) SIGRTMIN+5  40) SIGRTMIN+6  41) SIGRTMIN+7  42) SIGRTMIN+8
43) SIGRTMIN+9  44) SIGRTMIN+10 45) SIGRTMIN+11 46) SIGRTMIN+12 47) SIGRTMIN+13
48) SIGRTMIN+14 49) SIGRTMIN+15 50) SIGRTMAX-14 51) SIGRTMAX-13 52) SIGRTMAX-12
53) SIGRTMAX-11 54) SIGRTMAX-10 55) SIGRTMAX-9  56) SIGRTMAX-8  57) SIGRTMAX-7
58) SIGRTMAX-6  59) SIGRTMAX-5  60) SIGRTMAX-4  61) SIGRTMAX-3  62) SIGRTMAX-2
63) SIGRTMAX-1  64) SIGRTMAX
```
前面的数字代表的是信号值，SIGRTMIN到SIGRTMAX之间信号是实时信号。每个信号对应的都有默认动作，标准信号中大多数信号的默认动作都是终止+产生core文件，core文件包含程序出错的现场信息。

可以通过`ulimit -c`命令设置core文件的大小，这样如果程序中出现段错误，就会产生core文件。如果程序的时候加入-g调试选项，可以通过gdb查询core文件，分析错误产生的原因。

#### signal()

```
#include <signal.h>
typedef void (*sighandler_t)(int);
sighandler_t signal(int signum, sighandler_t handler);
```

signal()函数的作用是定义信号的行为。signum为信号值，handler是信号的回调函数。

**信号会打断阻塞的系统调用。**

#### 信号的不可靠

信号会打断阻塞的调用，返回EINTR的错误。

#### 可重入函数

什么是信号的可重入，就是信号的第一次调入还没有结束，就发送第二次调用。所有的系统调用都是可重入的，一部分库函数也是可重入的，比如说memcpy。如果一个函数提供了_r版本，那么这个函数的原函数一定不能出现再信号处理中。

#### 信号的响应过程

内核为每个进程维护两个位图，一个叫mask，一个叫pending。假如现在有个程序，功能是打印10个`*`，每隔1秒打印一个。程序中有一个main函数和一个handler函数，handler为信号处理函数。在main函数每隔一秒打印一个`*`，当按`ctrl+c`的时候，在信号处理函数handler中打印一个`!`。mask是信号屏蔽字，用来表示信号的状态，pending用来表示当前进程收到了哪些信号。mask位图的值一般情况下都是1，pending的值默认为0。信号从收到到响应有一个不可避免的延迟。

当进程因为时间片耗尽，保存执行现场后，从用户态切换到内核态，当前进程进入到一个等待队列，等待CPU的调度。直到当前进程重新获取CPU恢复执行，恢复执行时切换到用户态，通过`mask&pending`位运算来判断是否有信号，为0则无信号发生。当有信号发生时，会将pending中对应的位置为1，恢复调度的时候`mask&pending`运算就得到对应为1，说明有信号发生。信号发生时只时将pending中的位置为1，只有进程被中断打断，恢复执行时，才会检查中断位，才能发现是否有信号发生。当有信号发生时，先将对应的mask位和pending位置为0，将当前进程执行地址替换注册信号回调函数的地址，执行回调函数。信号处理函数处理完之后，切换为内核态，将进程的执行地址替换为原来的地址，将mask位恢复1。然后再从内核态变为内核态，做`mask&pending`的位运算，由于刚刚信号回调已经处理，进程的执行地址已经替换为最开始的执行地址，当前进程恢复执行中断前的代码。

标准信号的响应没有严格的顺序。

当设置信号处理方式为忽略时，就是将对应信号的mask位置为0，这样对应的信号就会被忽略。

#### 信号的常用函数

- kill()

- raise()

- alarm()

- pause()

- abort()

- system()

- sleep()

#### 信号集

#### 信号屏蔽字&pending集的处理

#### 扩展

- signsuspend()

- sigaction()

- setitimer()

#### 实时信号

### 线程