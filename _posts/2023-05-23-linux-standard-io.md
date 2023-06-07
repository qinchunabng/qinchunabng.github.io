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

#### fgets和fputs函数

fgets函数原型:
```
char *fgets(char *s, int size, FILE *stream);
```
fgets函数的作用是读取size-1个字符，或者读取到换行符（换行符也会读取到缓冲区中）或者文件结尾，读取的内容存入到s中，后面再加上'\0'结束符。

fputs函数原型：
```
 int fputs(const char *s, FILE *stream);
```
fputs函数的作用是把字符串s中的内如写入到stream文件流中。

用户fgets和fputs函数重写前面一个例子：
```
#include <stdio.h>
#include <stdlib.h>

#define BUFSIZE 1024

int main(int argc,char *argv[]){
    FILE *fps,*fpd;
    int c;
    char buf[BUFSIZE];

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
    
    while(fgets(buf,BUFSIZE,fps)!=NULL)
        fputs(buf,fpd);

    fclose(fps);
    fclose(fpd);
    return 0;
}
```

#### fread和fwrite函数

函数原型：
```
size_t fread(void *ptr, size_t size, size_t nmemb, FILE *stream);

size_t fwrite(const void *ptr, size_t size, size_t nmemb,
                     FILE *stream);
```
fread函数的作用是从stream文件流读取nmemb个成员，每个成员的大小为size大小到ptr中（总大小size*nmemb）。

fwrite函数的作用是将ptr中nmemb个size大小的成员写入到文件里stream中。

2个函数调用成功时，返回值为读取或写入成员的数量。这两个函数一般是用在读写多个固定大小得结构体变量时使用，需要主要的是，在读取的时候如果其中一个变量的数据出现错误会影响到后面的成员的读取。

#### atoi函数

```
 int atoi(const char *nptr);
```

atoi函数的作用是将一个给定的字符串转换为int。在转换的过程中，直到检查到字符串的结束符或者非数字字符，例如：
```
int i=atoi("123a456");
printf("%d\n",123);
```
结果为：123。


#### sprintf和snprintf函数

```
int sprintf(char *str, const char *format, ...);
```
sprintf函数的作用是将指定格式转换为字符串。示例：
```
char buf[1024];
int year=2023,month=6,day=5;
sprintf(buf,"%d-%d-%d", year, month, day);
printf("%s\n", buf);
```
输出结果：
```
2023-6-5
```

```
int snprintf(char *str, size_t size, const char *format, ...);
```
sprintf函数在转换没有指定字符串的大小，可能会导致内存的溢出。snprintf函数在sprintf函数基础增加了size参数，可以指定接受字符串的大小。

#### fseek、ftell和rewind函数

```
int fseek(FILE *stream, long offset, int whence);

long ftell(FILE *stream);

void rewind(FILE *stream);
```

fseek函数的作用是设置stream文件流的位置，offset为偏移量，whence为设置偏移量的位置。whence的取值：
- SEEK_SET：文件的开始位置
- SEEK_CUR：文件的当前位置
- SEEK_END：文件的结束位置
  
ftell函数的作用是获取文件流的当前位置。需要注意的是返回参数为long，但是long在C的标准中没有明确定义大小，如果是在32位机器上，long大小为4字节。但是ftell函数返回的位置只能是正数，也就是2^31，也就是2G，所以对应的在32位机器上fseek函数设置文件的不能超过2G。

rewind函数的作用是设置stream文件流指向文件开始的位置，相当于：
```
(void)fseek(stream,0,SEEK_SET);
```

fseek函数还有一个作用就是创建空洞文件，空洞文件就是文件内容都为0的文件。空洞文件的作用一般是在网络上下载文件的时候，根据下载的文件大小，先创建一个空洞文件占有磁盘空间，然后将文件分为多个块，下载的时候启动多个线程，每个线程写入一部分块的内容。


#### 输出缓冲区和fflush函数

看下面一个例子：
```
int main(){
    printf("Before while()");

    while(1);

    printf("After while()");
    exit(0);
}
```
按照代码逻辑，我们可能会认为这段代码的输出为：
```
Before while()
```
但是运行代码之后，实际上不会输出任何内容。原因是由于标准输出是行缓存模式，只有在遇到换行的时候，或者一行满了的时候才会刷新缓冲区。
所以上面代码如果要即时输出结果，可以修改为：
```
int main(){
    printf("Before while()\n");

    while(1);

    printf("After while()\n");
    exit(0);
}
```
输出结果：
```
Before while()
```

或者使用fflush函数主动刷新缓冲区。fflush函数原型：
```
int fflush(FILE *stream);
```
fflush函数的作用就是强制刷新stream的输出缓冲区的数据，如果stream为NULL，会刷新所有开启输出流。所以前面的代码需要也可以修改为：
```
int main(){
  printf("Before while()");
  fflush(stdout);

  while(1);

  printf("After while()");
  fflush(NULL);
}
```

缓冲区的作用：合并系统调用，在大多数情况是优点。

缓冲区的模式：
- 行缓冲：换行的时候刷新，满了的时候刷新，强制刷新（标准输出是这样的）
- 全缓冲模式：满了的时候刷新，强制刷新（默认，非终端设备除外）
- 无缓冲：如：stderr，需要立即输出的内容
#### fseeko和ftello函数

```
int fseeko(FILE *stream, off_t offset, int whence);

off_t ftello(FILE *stream);
```

fseeko和ftello函数与fseek和ftell函数的作用是一样的，不同的是fseeko函数offset参数和ftello返回值的类型改为off_t，为一个类型的别名。在一些系统中off_t和long都是32字节，但是可以在编译的时候通过参数`-D_FILE_OFFSET_BITS=64`指定off_t为8字节，也可以在makefile文件中添加`CFLAGS+=-D_FILE_OFFSET_BITS=64`来修改。但是这个两个函数是属于POSIX.1-2001, POSIX.1-2008, SUSv2的方言，移植性不太好。

#### getline函数

函数原型：
```
ssize_t getline(char **lineptr, size_t *n, FILE *stream);
```

getline函数的作用是从stream流中读取完整的一行数据到`*lineptr`所指向的地址中。如果`*lineptr`设置为NULL，`*n`设置为0，getline函数会给`*lineptr`分配内存并将读取的数据放到分配内存中，并设置`*n`为对应的大小。所以`*lineptr`在使用后，应该手动释放内存，即使getline函数调用失败。如果在调用函数前，`*lineptr`已经指定一块`*n`大小的内存，并且在getline调用过程中，`*lineptr`的大小不够，getline函数会使用realloc函数重新给`*lineptr`分配内存，更新`*lineptr`和`*n`的值。

#### 其他函数

```
int scanf(const char *format, ...);
int fscanf(FILE *stream, const char *format, ...);
int sscanf(const char *str, const char *format, ...);
```
- scanf函数的作用是从标准输入中读取内容到指定变量。
- fscanf函数的作用是从文件流中读取内容到指定变量。
- sscanf函数的作用是从字符串读取内容到指定变量。

注意：在指定读取变量为字符串的时候，因为无法预测输入内容的大小，可能会导致输入的内容超过字符串的大小，这是很危险的。

#### 临时文件

在写程序中，经常会用到一些临时文件，但是Linux是一个多任务系统，如何避免临时文件冲突？下面介绍两个函数：
```
char *tmpnam(char *s);
```
tmpnam函数的作用是获取一个有效的不存在的文件名。如果传入的参数s为NULL，文件名生成后存储在内部的静态缓冲区中，在下一次调用的时候会被覆盖。如果参数s不为NULL，生成的文件名被复制到s中，调用成功返回生成的文件名。

不推荐使用这个函数，因为在高并发请求，如果一个线程或者进程获取一个文件，然后再去创建这个文件，在这个过程中文件还没创建，另外一个线程可能会拿到同样的文件名，也是创建或打开这个文件，这个时候产生的临时文件就冲突了。出现这个问题的原因是因为获取文件名和创建文件不是一个原子的过程。所以推荐使用下面这个函数替代：
```
FILE *tmpfile(void);
```
tmpfile函数的作用是打开一个唯一的临时，以二进制读写模式（w+b）打开，文件将会在关闭或者程序停止时删除。如果函数调用成功，返回指向临时文件的文件流。如果无法生成一个唯一的文件或者文件打开失败返回NULL，并且errno设置为对应的错误码。