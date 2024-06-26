---
layout: post
title: Linux并发——线程
categories: [Linux]
description: Linux并发——线程
keywords: Linux, C
---


### 线程

#### 线程的概念

线程就是一个正在运行的函数。一个线程同时只能运行一个函数。多个线程共享内存空间。

线程有多个不同的标准，目前用的比较多的是POSIX线程。POSIX线程是一套标准，而不是实现。另外还有openmp线程。

可以通过`ps axm`和`ps ax -L`查看进程和线程之间的关系。

线程标识：pthread_t。由于POSIX线程是一个标准，各个系统实现不一样，所以pthread_t具体是什么类型是不清楚的的。所以如果要判断两个线程是否一样，需要使用下面的函数：
```
#include <pthread.h>

int pthread_equal(pthread_t t1, pthread_t t2);
```
两个线程相对，返回0，否则返回非0。

获取当前线程：
```
#include <pthread.h>

pthread_t pthread_self(void);
```

使用以上两个函数，编译和链接的时候要加上-pthread标识。

#### 线程的创建、终止、清理和取消

##### 线程的创建

```
 #include <pthread.h>

int pthread_create(pthread_t *thread, const pthread_attr_t *attr,
                          void *(*start_routine) (void *), void *arg);
```

pthread_create()函数的作用是创建一个线程。thread是一个指向线程标识的指针，创建线程后将会将线程标识回填。attr是线程属性，如果传NULL，使用默认属性。start_routine是一个入参和返回值都为void\*类型函数地址，这个函数线程将要执行的函数。arg是传给线程执行函数的参数。

pthread_create()调用成功返回0，失败返回错误码。

##### 线程终止

三种方式：
1) 线程从启动例程返回，返回值就是线程的退出码
2) 线程可以被同一进程中的其他线程取消
3) 线程调用pthread_exit()函数

```
#include <pthread.h>

int pthread_join(pthread_t thread, void **retval);
```

pthread_join()函数类似于wait()函数，用于给thread收尸。thread是对应线程标识，retval是线程的返回值，返回值的内容是pthread_exit()函数设置的。如果线程被取消，PTHREAD_CANCELED会放入retval指向的空间。

##### 栈的清理

```
#include <pthread.h>

void pthread_cleanup_push(void (*routine)(void *),
                                 void *arg);
void pthread_cleanup_pop(int execute);
```

pthread_cleanup_push()函数作用是向当前线程的清理handler栈中注册钩子函数，routine为函数地址，arg为函数的参数，可以在清理函数中进行一些锁的释放等操作。

pthread_cleanup_pop()函数作用是从当前函数的清理handler中弹出注册的函数，如果execute参数不为0，弹出的清理函数会被执行。

在以下几种情况下，注册的清理函数会被弹出栈并且执行：
1. 如果线程被取消，栈中函数会弹出并执行，执行顺序与注册顺序相反。
2. 线程被pthread_exit终止，清理函数会弹出并执行。
3. 执行pthread_cleanup_pop()函数，并且execute非0，栈顶部的函数会弹出并且执行。

#### 线程的同步

#### 线程的属性

#### 重入/线程与信号/线程与fork

#### 