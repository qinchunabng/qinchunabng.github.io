---
layout: post
title: Docker的基本实现原因
categories: [Docker]
description: Docker的基本实现原因
keywords: Docker
---

### 实现原理

Docker虚拟化需要解决的核心问题：资源隔离和资源限制

- 虚拟机硬件虚拟化技术，通过hypervisor层实现对资源的彻底隔离。
- 容器则是通过操作系统级别的虚拟化，利用的是内核的Cgroup和Namespace特性，此功能完全通过软件实现。

#### Namespace资源隔离

<a id="namespace">命名空间</a>是全局资源的一种抽象，将资源放到不同的命名空间中，各个命名空间的资源是相互隔离的。
|分类|系统调用参数|相关内核版本|
|--|--|--|
|Mount namespaces|CLONE_NEWNS|Linux2.4.19|
|UTS namespaces|CLONE_NEWUTS|Linux2.4.19|
|IPC namespaces|CLONE_NEWIPC|Linux2.4.19|
|PID namespaces|CLONE_NEWPID|Linux2.4.24|
|Network namespaces|CLONE_NEWNET|始于Linux2.6.24完成与Linux2.6.29|
|User namespaces|CLONE_NEWUSER|始于Linux2.6.23完成与Linux3.8|

查看当前进程的namespace:
```
$ ls -l /proc/$$/ns
total 0
lrwxrwxrwx 1 qcb qcb 0 Mar  6 22:29 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 qcb qcb 0 Mar  6 22:29 ipc -> 'ipc:[4026532211]'
lrwxrwxrwx 1 qcb qcb 0 Mar  6 22:29 mnt -> 'mnt:[4026532209]'
lrwxrwxrwx 1 qcb qcb 0 Mar  6 22:29 net -> 'net:[4026531992]'
lrwxrwxrwx 1 qcb qcb 0 Mar  6 22:29 pid -> 'pid:[4026532212]'
lrwxrwxrwx 1 qcb qcb 0 Mar  6 22:29 pid_for_children -> 'pid:[4026532212]'
lrwxrwxrwx 1 qcb qcb 0 Mar  6 22:29 user -> 'user:[4026531837]'
lrwxrwxrwx 1 qcb qcb 0 Mar  6 22:29 uts -> 'uts:[4026532210]'
```

我们知道，docker容器对于操作系统来讲其实是一个进程，我们可以通过原始的方式模拟一下容器实现资源隔离的基本原理：

在linux操作系统中，通常可以用过clone()实现创建进程的系统调用，函数的原型如下：
```
 int clone(int (*fn)(void *), void *stack, int flags, void *arg, ...);
```
- fn:创建的子进程要执行的函数。
- stack:子进程栈空间的地址。
- flags:创建子进程的标识参数，指定父子进程之间哪些资源共享，取值为[命名空间](#namespace)的系统调用参数。
- arg:fn函数的函数。