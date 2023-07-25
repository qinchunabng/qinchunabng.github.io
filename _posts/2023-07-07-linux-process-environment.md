---
layout: post
title: 进程环境
categories: [Linux]
description: Linux环境
keywords: Linux, C
---

## 进程环境

### main函数

```
int main(int argc,char *argv[]);
```

### 进程终止

#### 正常终止

- 从main函数返回

```
int main(){
    printf("Hello\n");
    return 0;
}
```
main函数的返回值是给父进程看的。如果我们编译这个程序并执行，因为我们是从命令行中调用该程序，所以它的父进程就是shell进程，可以通过
`echo $?`打印上一条语句的执行状态，得到结果就是0。

- 调用exit函数

- 调用_exit或_Exit函数
  
- 最后一个线程从启动例程返回
  
- 最后一个线程调用pthread_exit

#### 异常终止

- 调用abort函数

- 接到一个信号并终止
  
- 最后一个线程对取消请求作出响应

正常终止和异常终止的区别在于，正常终止，程序会刷新流，会调用钩子函数，而异常终止不会。如果程序是异常终止，可能会导致文件不关闭，通知未发送，锁未释放等。

#### exit函数

```
void exit(int status);
```

exit函数会导致一个进程正常终止，并将`status & 0377`返回给父进程。在shell中，可以通过$?获取上一个执行进程的返回值。

通过exit函数终止一个进程后，通过atexit和on_exit注册的所有函数会以注册时相反的顺序调用。

atexit通常称为钩子函数。atexit函数的原型：
```
int atexit(void (*function)(void));
```

示例：
```
#include <stdlib.h>
#include <stdio.h>

static void f1(){
    puts("f1 is running.");
}

static void f2(){
    puts("f2 is running.");
}

static void f3(){
    puts("f3 is running.");
}

int main(){
    puts("Begin!");

    atexit(f1);
    atexit(f2);
    atexit(f3);
    
    puts("End!");
    exit(0);
}
```
输出结果：
```
Begin!
End!
f3 is running.
f2 is running.
f1 is running.
```

exit函数的作用与_exit或_Exit函数的作用相同，不同的是exit是库函数，而_exit和_Exit是系统调用。