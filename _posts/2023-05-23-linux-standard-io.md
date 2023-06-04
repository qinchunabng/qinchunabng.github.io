---
layout: post
title: Linux系统开发之标准I/O
categories: [Linux]
description: 标准I/O
keywords: C, Linux编程
---

## 标准I/O

I/O即输入和输出，I/O是一切实现的基础，如果没有I/O数据就无法保存。I/O分为两种：stdio标准IO和sysio系统调用IO（文件IO）。在两种都可以使用的时候，优先使用标准IO。

### 系统调用IO和标准IO的区别

我们的程序一般都运行在用户态，如果要与内核态进行对话时，需要通过提供系统调用函数。如果系统不一样，提供的系统函数就不一样，就会给用户造成困扰。这个时候标准就出现了，来解决不同系统的系统调用不一样的问题。标准IO提供标准，各个平台各自去实现。所以在标准IO和系统调用IO都可用的时候，优先使用标准，移植性好。

### 标准IO

标准IO（stdio）常见函数：
```
fopen();
fclose();
fgetc();
fgets();
fputs();
fread();
fwrite();

printf();
scanf();

fseek();
ftell();
rewind();

fflush();
```

了解一个函数的使用方法，可以使用man手册查询。比如要查看fopen使用方法，可以使用如下命令：
```
man fopen
```
man手册的第1章是基本的命令，第2章是系统调用，第3章是标准库函数，第7章讲机制，可以通过如下命令指定第几章：
```
man 3 fopen
```
以上命令明确指定从第3章查询fopen函数。

#### fopen函数

```
FILE *fopen(const char *pathname, const char *mode);
```

fopen函数有两个参数：pathname和mode。pathname为文件路径，mode为打开文件的模式，取值为：
- r。只读模式打开文件。文件流指向文件开始的位置。
- r+。以读写模式打开文件。文件流指向文件开始的位置。
- w。清空文件内容，创建新文件来写入。文件流指向文件开始的位置。
- w+。以读写的模式打开文件。如果文件不存在则创建文件，否则清空文件。文件流指向文件开始的位置。
- a。追加的模式打开文件（向文件尾部写入）。如果文件不存咋则创建。文件流指向文件结尾。
- a+。读模式打开文件，并且可以向文件尾部追加写入。如果文件不存在则创建。如果是读操作，读取的起始位置是是在文件的开始位置，如果是写操作，写入内容总是追加文件的尾部。

还有一个特殊的字符b，在以上取值后面加上b，表示打开的是字节文件，但是b字符在所有的POSIX系统，包括Linux，会被忽略。因为Linux上打开的都是字节流，但是在其他的系统比如Windows上，是区分字符文件和字节文件的，如果打开的是字节文件，需要加b。另外mode如果内容超过2个或3个字符，只会读取前面几个有效字符。例如：mode传read，有效值为r。

pathname和mode都是常量指向，表示参数的值在函数执行过程中不会发生改变。

函数调用成功会返回FILE的指针，否则返回NULL，并且errno设置为设置为对应错误。

errno是一个全局变量，如果调用的函数出现错误，会把错误值放在errno中。可以在/usr/include/asm-generic/errno.h和/usr/include/asm-generic/errno-base.h中查看到errno有哪些定义值。    

#### fclose函数

我们来分析fopen的返回值，指向的空间是栈、静态区或堆上。
- 在栈上。栈空间的内存只在函数作用域内有效，所以方法执行完之后会被释放，所以不可能是在栈上。
- 在静态区。静态区内存有个特点就是后面调用的函数的内容会覆盖前面的内容，如果是在静态区，多次调用fopen函数，指针指向的内容只能是最后一次调用的结果，所以不可能是静态区。
- 在堆上。如果是在堆上，需要通过malloc函数分配内存，函数执行完返回指针，指向的内存并不会被回收，也不存在静态区内存被覆盖的问题，索引fopen返回指针指向的内存应该是在堆上。
  
如果fopen返回的指针指向的内存是在堆上，不会自动释放，那么就需要手动释放。那么就出现fclose函数，fclose函数就是用来释放fopen的申请的内存。所以，我们通过观察一个返回指针的函数是否提供逆操作来判断返回指针指向的内存是否在堆上。

代码示例：
```
int main()
{
  FILE *fp;
  fp = fopen("tmp", "r");
  if (fp == NULL)
  {
    perror("fopen()");
    exit(1);
  }
  puts("OK");
  fclose(fp);
  return 0;
}
```
如果文件不存在，输出结果：
```
fopen(): No such file or director
```
perror函数的作用就是将最近一次系统调用或者库函数的错误码转为错误描述，并拼接上perror函数传入的参数，并打印。perror也可以用如下代码替换：
```
fprintf(stderr, "fopen(): %s\n", strerror(errno));
```
strerror函数的作用就是获取错误码对应的错误描述。

如果打开文件不关闭，打开的文件多了会报错，看如下代码：
```
int main()
{
  int count=0;
  FILE *fp=NULL;
  while(1){
    fp=fopen("/tmp","r");
    if(fp==NULL){
      perror("fopen()");
      break;
    }
    count++;
  }
  printf("count=%d",count);
}
```
输出结果：
```
fopen(): Too many open files
count=1021
```
当打开1021个文件的时候，报错了。因为每个进程默认打开的最大文件描述符的数量为1024，每个进程创建的时候，默认会打开stdin、stdout、stderr三个流，加起来就是1024。这个大小可以通过如下命令查看：
```
qcb@qincb:/mnt/c/Users/37341$ ulimit -a
real-time non-blocking time  (microseconds, -R) unlimited
core file size              (blocks, -c) unlimited
data seg size               (kbytes, -d) unlimited
scheduling priority                 (-e) 0
file size                   (blocks, -f) unlimited
pending signals                     (-i) 63459
max locked memory           (kbytes, -l) 64
max memory size             (kbytes, -m) unlimited
open files                          (-n) 1024
pipe size                (512 bytes, -p) 8
POSIX message queues         (bytes, -q) 819200
real-time priority                  (-r) 0
stack size                  (kbytes, -s) 8192
cpu time                   (seconds, -t) unlimited
max user processes                  (-u) 63459
virtual memory              (kbytes, -v) unlimited
file locks                          (-x) unlimited
```
open files即最大打开文件数，可以通过`ulimit -n`命令修改。

#### fgetc和fputc函数

fgetc和fputc的函数原型：
```
int fgetc(FILE *stream);
int fputc(int c, FILE *stream);
```
fgetc函数从文件流中读取一个字符，读取成功返回一个转换为int的unsigned char，读取失败返回EOF或者一个错误码。

fputc函数的作用是将c转换为unsinged char，并且写入到文件流。写入成功返回写入的字符，由unsigned char转为int，写入失败返回EOF或者错误码。

下面通过一个案例，演示fopen、fgetc、fputc、fclose函数的使用，通过在这个程序可以将一个文件的内容拷贝到另外一个文件中：
```
#include <stdio.h>
#include <stdlib.h>

int main(int argc,char *argv[]){
    FILE *fps,*fpd;
    int c;

    if(argc < 3){
        fprintf(stderr,"Usage:%s <src_file> <dest_file>\n",argv[0]);
        exit(1);
    }

    fps=fopen(argv[1],"r");
    if(fps==NULL){
        perror("fopen()");
        exit(1);
    }

    fpd=fopen(argv[2],"w");
    if(fpd==NULL){
        //如果目标文件打开失败，记得关闭第一个文件，否则会出现内存泄漏
        fclose(fps);
        perror("fopen()");
        exit(1);
    }
    
    while(1){
        c=fgetc(fps);
        if(c==EOF){
            break;
        }
        fputc(c,fpd);
    }

    fclose(fps);
    fclose(fpd);
    return 0;
}
```