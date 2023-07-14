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
