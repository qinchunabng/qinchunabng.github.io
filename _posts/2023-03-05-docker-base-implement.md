---
layout: post
title: Docker的基本实现原理
categories: [Docker]
description: Docker的基本实现原因
keywords: Docker
---

## 实现原理

Docker虚拟化需要解决的核心问题：资源隔离和资源限制

- 虚拟机硬件虚拟化技术，通过hypervisor层实现对资源的彻底隔离。
- 容器则是通过操作系统级别的虚拟化，利用的是内核的Cgroup和Namespace特性，此功能完全通过软件实现。

### Namespace资源隔离

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

示例一：实现进程独立的UTS空间
```
#define __GNU_SOURCE
#include <stdlib.h>
#include <stdio.h>
#include <linux/sched.h>
#include <sys/wait.h>
#include <signal.h>
#include <unistd.h>

#define STACK_SIZE (1024*1024)

static char container_stack[STACK_SIZE];
char * const container_args[]={"/bin/bash",NULL};

int container_main(void* arg){
    printf("Container - inside the container!\n");
    sethostname("container",10);
    execv(container_args[0], container_args);
    printf("something is wrong!\n");
    return -1;
}

int main(){
    int container_id;
    printf("Parent - start a container!\n");

    container_id = clone(container_main, container_stack + STACK_SIZE, CLONE_NEWUTS | SIGCHLD, NULL);
    waitpid(container_id, NULL, 0);
    printf("Parent - container stopped!\n");
    exit(0);
}
```
执行编译并测试：
```
root@qincb:/mnt/d/Workspace/C/c_learning/clone# ./clone
Parent - start a container!
Container - inside the container!
root@container:/mnt/d/Workspace/C/c_learning/clone# hostname
container
root@container:/mnt/d/Workspace/C/c_learning/clone# echo $$
61
root@container:/mnt/d/Workspace/C/c_learning/clone# ls -l /proc/61/ns
total 0
lrwxrwxrwx 1 root root 0 Mar  8 09:46 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Mar  8 09:46 ipc -> 'ipc:[4026532211]'
lrwxrwxrwx 1 root root 0 Mar  8 09:46 mnt -> 'mnt:[4026532209]'
lrwxrwxrwx 1 root root 0 Mar  8 09:46 net -> 'net:[4026531992]'
lrwxrwxrwx 1 root root 0 Mar  8 09:46 pid -> 'pid:[4026532212]'
lrwxrwxrwx 1 root root 0 Mar  8 09:46 pid_for_children -> 'pid:[4026532212]'
lrwxrwxrwx 1 root root 0 Mar  8 09:46 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Mar  8 09:46 uts -> 'uts:[4026532288]'
```
新启动一个终端：
```
qcb@qincb:/mnt/d/Workspace/C/c_learning$ hostname
qincb
qcb@qincb:/mnt/d/Workspace/C/c_learning$ echo $$
72
qcb@qincb:/mnt/d/Workspace/C/c_learning$ ls -l /proc/72/ns
total 0
lrwxrwxrwx 1 qcb qcb 0 Mar  8 09:49 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 qcb qcb 0 Mar  8 09:49 ipc -> 'ipc:[4026532211]'
lrwxrwxrwx 1 qcb qcb 0 Mar  8 09:49 mnt -> 'mnt:[4026532209]'
lrwxrwxrwx 1 qcb qcb 0 Mar  8 09:49 net -> 'net:[4026531992]'
lrwxrwxrwx 1 qcb qcb 0 Mar  8 09:49 pid -> 'pid:[4026532212]'
lrwxrwxrwx 1 qcb qcb 0 Mar  8 09:49 pid_for_children -> 'pid:[4026532212]'
lrwxrwxrwx 1 qcb qcb 0 Mar  8 09:49 user -> 'user:[4026531837]'
lrwxrwxrwx 1 qcb qcb 0 Mar  8 09:49 uts -> 'uts:[4026532210]'
```
对比clone的子进程和外部的进程，子进程修改了hostname，但是外部进程的中的hostname还是原来的，并且子进程uts命令空间和外部进程的不一样，说明通过命名空间实现了资源隔离。

综上：通俗来讲，docker在启动一个容器时，会调用Linux Kernel Namespace的接口，来创建一块虚拟空间，可以支持下面这几种（可以随意设置），docker默认都设置：
- pid: 用于进程隔离（PID: 进程ID）
- net: 管理网络接口（NET: 网络）
- ipc: 管理对IPC资源的使用（IPC: 进程间通信（信号量/消息队列/共享内存））
- mnt: 管理文件系统挂载点（MNT: 挂载）
- uts: 隔离主机名和域名
- user: 隔离用户和用户组
  
### CGroup资源限制

通过namespace可以保证容器间的隔离，但是无法控制每个容器可以占用多少资源，如果其中一个容器正在执行CPU密集型的任务，就会影响其他容器中任务的性能和执行效率，导致多个容器相互影响和资源抢占。如何对容器使用的资源进行限制就成为进程虚拟资源隔离之后的主要问题。

![docker共享资源](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/docker/docker_shared_resources.png?raw=true)

Control Groups（简称CGroups）能隔离宿主机上物理资源，例如CPU、内存、硬盘I/O和网络带宽。每个CGroup都是一组被相同标准和参数限制的进程。而我们需要做的其实就是把容器的这个进程加入到指定的CGroup中。更多详细信息`man cgroups`参考man手册。

### UnionFS联合文件系统

Linux Namespace和CGroup分别解决资源隔离和资源限制，那么容器是很轻量的，通常每台机器上可以运行几十上百个容器，这些容器是公用一个image，还是各自将这个image各自复制一份，然后各自独立运行呢？如果每个容器之间都是全量的文件系统拷贝，那么至少将导致以下问题：

- 运行容器的速度会变慢
- 容器和镜像对宿主机的磁盘空间的压力

怎么解决这个，Docker的存储驱动
- 镜像分层存储
- UnionFS
  
Docker镜像是由一系列的层组成的，每层代表Dockerfile中的一条指令，比如下面的Dockerfile：
```
FROM ubuntu:15.04
COPY . /app
RUN make /app
CMD python /app/app.py
```
这里的Dockerfile包含4条指令，每条指令会创建一层，下图为上述Dockerfile构建出来的镜像容器层结构：

![容器层结构](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/docker/docker_container_layer.png?raw=true)

镜像就像是由这些层一层层堆叠起来的，镜像中这些层都是只读的，当我们运行容器时，就可以在这些层基础上添加新的可写层，也就是我们通常说的*容器层*，对于运行的容器所作的更改（比如写入新文件，修改删除现有的文件）都将写入这个容器层。

对于容器层的操作，主要时利用的写时复制（copy-on-write）技术，表示只有在写的时候才需要复制，这是针对已有文件的修改场景。copy-on-write技术让所有的容器共享image的文件系统，所有的数据都从image读取，只有要对文件进行写操作时，才从image里把要写的文件复制到自己的文件系统进行修改。所以无论有多少个容器都是共享一个image，所有的写操作都是复制image到自己的文件系统的副本进行的，并不会修改image的源文件，且多个容器操作一个文件，会在每个容器的文件系统生成一个副本，每个容器都是修改的自己的副本，相互隔离，互不影响。使用copy-on-write可以有效的提高磁盘利用效率。


