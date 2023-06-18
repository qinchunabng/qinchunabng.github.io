---
layout: post
title: 系统调用I/O
categories: [Linux]
description: 系统调用I/O
keywords: C, Linux编程
---

## 系统调用I/O

### 文件描述符实现原理

在标准I/O中FILE贯穿始终，FILE其实是一个结构体，结构体包含了操作文件所有的属性。而文件描述符，就是在系统创建一个数组，在打开一个文件时，将创建的一个包含操作文件所需要的所有属性的结构体，将该结构体的起始位置存放到数组中的某个位置，程序返回的时这个数组索引的下标。所以文件描述符的实质是一个int，因为它是数组索引的下标。这个存放文件描述符的数组大小就是一个程序能够打开的文件数的大小，这个数组默认的大小为1024，可以通过ulimit修改默认大小，每个进程的文件描述符数组0、1和2的位置默认分别对应标准输入（stdio）、标准输出（stdout）和标准出错（stderr）。新打开一个文件时，默认使用当时可使用范围内最小的一个。这个数组是存在进程空间中，每个进程都有这样的一个数组。close时候会释放对应文件描述符对应结构体，并且将对应下标置为空。同一个文件打开多次，会长产生多个文件描述符，产生多个文件操作的结构体。

![文件描述符示意图](https://github.com/qinchunabng/qinchunabng.github.io/blob/master/images/posts/linux/%E6%96%87%E4%BB%B6%E6%8F%8F%E8%BF%B0%E7%AC%A6%E7%A4%BA%E6%84%8F%E5%9B%BE.png?raw=true)

### open和close函数

```
int open(const char *pathname, int flags);
int open(const char *pathname, int flags, mode_t mode);
```
open函数发起一个系统调用，打开pathname参数指定的路径。如果对应的路径的文件不存在，可以在flags参数设置O_CREAT来创建文件。open函数的返回值为是文件描述符。flags参数必须为三种访问模式之一：O_RDONLY、O_WRONLY、O_RDWR，分别为只读、只写、读写模式打开文件。另外，多个文件创建flags和文件状态flags可以按照按位或的方式设置到flags参数中。文件创建flags影响文件的打开操作，文件状态flags影响之后的I/O操作。文件创建flags包含了O_CLOEXEC、O_CREAT、O_DIRECTORY、OEXCL、O_NOCITY、O_NOFOLLOW、O_TMPFILE和O_TRUNC。flags一共有很多取值，详细通过`man open`查看手册。 第二个open函数的mode参数是创建文件的权限，最终的权限值是~(mode&u_mask)。

```
int close(int fd);
```
close函数的作用是关闭一个文件描述符，调用成功返回0，错误返回-1，并且将errno设置为对应的错误码。

### read、write和lseek函数

```
ssize_t read(int fd, void *buf, size_t count);
```
read函数的作用是文件描述符fd读取count个字节的内容到buf中。函数调用成功返回值表示读取到的字节数，返回值为0表示读取到文件尾，发生错误时返回值为-1，并且errno设置为对应错误码。需要注意的read函数的返回值并不一定会等于参数count，在读取文件尾或者读取发生了中断，可能会导致实际读取的字节数小于count。

```
ssize_t write(int fd, const void *buf, size_t count);
```
write函数的作用是从buf中写入count个字节到文件描述符fd所指向的文件中。函数调用成功时，返回值为写入的字节数，调用失败时返回值为-1，并且errno设置为对应的错误码。需要注意的是，实际写入的字节数可能小于count，例如硬盘上没有足够的空间写入所有数据，或者写入时被某个信号中断。

```
off_t lseek(int fd, off_t offset, int whence);
```
lseek函数的作用为设置为文件的偏移量。fd为指向文件的文件描述符，offset为文件的偏移量，whence参数定义修改偏移量的行为，取值有以下几种：
- SEEK_SET
  
  设置文件的偏移量为offset个字节。

- SEEK_CUR

  设置文件的偏移量为当前位置加上offset个字节。

- SEEK_END

  设置文件的偏移量为文件结尾加上offset个字节。

函数调用成功返回值为从文件开始到offset位置的处的字节数，失败返回-1，errno设置为对应的错误码。

下面用一个例子实现用系统调用I/O实现文件复制：
```
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

#define BUF_SIZE 1024

int main(int argc,char **argv){
    int sfd,dfd;
    char buf[BUF_SIZE];
    int len,ret,pos;
    if(argc < 3){
        fprintf(stderr, "Usage...\n");
        exit(1);
    }

    sfd = open(argv[1],O_RDONLY);
    if(sfd < 0){
        perror("open()");
        exit(1);
    }
    dfd = open(argv[2],O_WRONLY|O_CREAT|O_TRUNC,0600);
    if(sfd < 0){
        close(sfd);
        perror("open()");
        exit(1);
    }

    while(1){
        len=read(sfd,buf,BUF_SIZE);
        if(len<0){
            perror("read()");
            break;
        }
        if(len==0)
            break;

        pos=0;
        while(len>0){
            ret=write(dfd,buf+pos,len);
            if(ret<0){
                perror("write()");
                exit(1);
            }
            len-=ret;
        }
        
    }

    close(sfd);
    close(dfd);

    exit(0);
}
```


### 系统调用I/O和标准I/O的区别

标准I/O有缓冲区，写入并不是真正的写入到磁盘，而是是写入了缓冲区，除非主动调用fflush，否则并不是每次都会执行真正的I/O操作。而系统调用I/O，没调用一次都会真正执行一次I/O，从用户太切换到内核态执行一次，实时性较高。另外需要注意的是标准I/O和系统调用不可混用。

下面通过一个例子说明系统调用I/O和标准I/O的区别：
```
int main()
{
  putchar('a');
  write(1,"b",1);

  putchar('a');
  write(1,"b",1);

  putchar('a');
  write(1,"b",1);
}
```
按照代码执行的顺序，输出结果应该是ababab，但是实际输出结果为：bbbaaa。就是因为系统调用I/O是立即执行的，而标准I/O是先写入缓冲区。另外还可以通过strace命令查看上面代码最终执行逻辑，执行命令：
```
strace ./main
```
输出结果：
```
execve("./main", ["./main"], 0x7ffce3a301e0 /* 20 vars */) = 0
brk(NULL)                               = 0x558cc9fa5000
arch_prctl(0x3001 /* ARCH_??? */, 0x7fff9ad70f90) = -1 EINVAL (Invalid argument)
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f8d20683000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=18395, ...}, AT_EMPTY_PATH) = 0
mmap(NULL, 18395, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f8d2067e000
close(3)                                = 0
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0P\237\2\0\0\0\0\0"..., 832) = 832
pread64(3, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 784, 64) = 784
pread64(3, "\4\0\0\0 \0\0\0\5\0\0\0GNU\0\2\0\0\300\4\0\0\0\3\0\0\0\0\0\0\0"..., 48, 848) = 48
pread64(3, "\4\0\0\0\24\0\0\0\3\0\0\0GNU\0i8\235HZ\227\223\333\350s\360\352,\223\340."..., 68, 896) = 68
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=2216304, ...}, AT_EMPTY_PATH) = 0
pread64(3, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 784, 64) = 784
mmap(NULL, 2260560, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f8d20456000
mmap(0x7f8d2047e000, 1658880, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x28000) = 0x7f8d2047e000
mmap(0x7f8d20613000, 360448, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1bd000) = 0x7f8d20613000
mmap(0x7f8d2066b000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x214000) = 0x7f8d2066b000
mmap(0x7f8d20671000, 52816, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f8d20671000
close(3)                                = 0
mmap(NULL, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f8d20453000
arch_prctl(ARCH_SET_FS, 0x7f8d20453740) = 0
set_tid_address(0x7f8d20453a10)         = 396
set_robust_list(0x7f8d20453a20, 24)     = 0
rseq(0x7f8d204540e0, 0x20, 0, 0x53053053) = 0
mprotect(0x7f8d2066b000, 16384, PROT_READ) = 0
mprotect(0x558cc9226000, 4096, PROT_READ) = 0
mprotect(0x7f8d206bd000, 8192, PROT_READ) = 0
prlimit64(0, RLIMIT_STACK, NULL, {rlim_cur=8192*1024, rlim_max=RLIM64_INFINITY}) = 0
munmap(0x7f8d2067e000, 18395)           = 0
newfstatat(1, "", {st_mode=S_IFCHR|0620, st_rdev=makedev(0x88, 0x1), ...}, AT_EMPTY_PATH) = 0
getrandom("\x68\xd8\x52\x1a\xc3\x54\x84\xd5", 8, GRND_NONBLOCK) = 8
brk(NULL)                               = 0x558cc9fa5000
brk(0x558cc9fc6000)                     = 0x558cc9fc6000
write(1, "b", 1b)                        = 1
write(1, "b", 1b)                        = 1
write(1, "b", 1b)                        = 1
write(1, "aaa", 3aaa)                      = 3
exit_group(0)                           = ?
+++ exited with 0 +++
```
上面的代码实际执行过程是调用三次系统调用输出b，而三次标准I/O输出实际合并为一个系统调用I/O一次输出了aaa。