---
layout: post
title: Linux的文件系统
categories: [Linux]
description: Linux的文件系统
keywords: Linux, C
---

## 文件和目录

### 获取文件属性

```
int stat(const char *pathname, struct stat *statbuf);
int fstat(int fd, struct stat *statbuf);
int lstat(const char *pathname, struct stat *statbuf);
```
这几个函数的作用都是获取文件属性相关的内容，stat()函数参数是文件路径，fstat()函数参数是文件描述符，lstat()函数是获取连接文件的属性。

这些函数获取文件的属性信息，回填到`struct stat *statbuf`中。结构体的stat的定义：
```
 struct stat {
               dev_t     st_dev;         /* ID of device containing file */
               ino_t     st_ino;         /* Inode number */
               mode_t    st_mode;        /* File type and mode */
               nlink_t   st_nlink;       /* Number of hard links */
               uid_t     st_uid;         /* User ID of owner */
               gid_t     st_gid;         /* Group ID of owner */
               dev_t     st_rdev;        /* Device ID (if special file) */
               off_t     st_size;        /* Total size, in bytes */
               blksize_t st_blksize;     /* Block size for filesystem I/O */
               blkcnt_t  st_blocks;      /* Number of 512B blocks allocated */

               /* Since Linux 2.6, the kernel supports nanosecond
                  precision for the following timestamp fields.
                  For the details before Linux 2.6, see NOTES. */

               struct timespec st_atim;  /* Time of last access */
               struct timespec st_mtim;  /* Time of last modification */
               struct timespec st_ctim;  /* Time of last status change */

           #define st_atime st_atim.tv_sec      /* Backward compatibility */
           #define st_mtime st_mtim.tv_sec
           #define st_ctime st_ctim.tv_sec
           };
```
各个字段的含义：
- t_dev是包含当前问的设备ID。
- st_ino是文件的inode号。
- st_mode是文件的权限信息。
- st_nlink是文件的硬链接数。
- st_uid是文件拥有者的用户ID。
- st_gid是文件有着的组ID。
- st_rdev是设备ID。
- st_size是文件总大小，单位字节。
- st_blksize是文件系统的块大小。
- st_blocks是当前文件占用512字节大小块的数量。
- st_atim是最近一次的访问时间。
- st_mtim是最近一次的修改时间。
- st_ctim是最近一次状态变化的时间。

st_size是文件的大小，实际占用的磁盘空间为`st_blksize*st_blocks`。

st_mode包含了两部分，一部分是文件的类型，一部分是文件的权限。系统提供有些宏来检测文件的文件的类型：
- S_ISREG(m): 测试是否是常规文件
- S_ISDIR(m): 测试是否是目录
- S_ISCHR(m): 测试是否是字符设备
- S_ISBLK(m): 测试是否是块设备
- S_ISFIFO(m): 测试是否是管道
- S_ISLNK(m): 测试是否是符号链接
- S_ISSOCK(m): 测试是否是socket

stat: 通过文件路径获取属性，面对符号链接时获取的是所指向的目标目标文件的属性。
fstat: 通过文件描述获取属性。
lstat：面对符号链接文件，获取的是符号链接的文件。

### 文件访问权限

### umask

### chmod,fchmod

### 文件权限的更改/管理

### 粘住位

### 文件系统：FAT,UFS

文件系统：文件或数据的存储和管理。

FAT16/32实质是静态存储的单链表。结构体
```
struct {
   int next[N];
   char data[N][size];
};
```
![FAT文件存储结构示意图]()

FAT文件存储分为两个部分：next和data。data用来存储文件内容，next存储文件的索引下标，如上图所示，next数组种文件下标形成一个单链表。next数组中下标表示data中存储有数据的下标，文件引用为next数组中首个文件下标，next数组中对应下标的元素存储下一个存储文件内容的下标，最后一个next文件索引内容为NULL。最大文件的大小取决N的大小。FAT文件的最大缺陷是，其实现为单链表，无法向回查找。

### 硬链接，符号链接

硬链接是目录项的同义词，硬链接是指向目录项的一个指针，符号链接inode号与原文件是一样的。硬链接删除原文件，硬链接仍然可以使用。建立硬链接有限制：不能给分区建立，不能给目录建立。

符号链接类似windows下快捷方式，符号链接占用的空间很小，与原文件是两个不通的文件，有各自的属性。删除原文件会导致链接文件失效。符号链接的优点：可以跨分区，可以给目录建立。

```
#include <unistd.h>

int link(const char *oldpath, const char *newpath);
int unlink(const char *pathname);
```
link()函数的作用是创建一个硬链接指向一个存在的文件。硬链接和原文件指向的是同一个文件，我们无法通过属性区分哪个是源文件。

unlink()函数的作用删除一个文件，如果文件删除文件是最后一个指向文件链接，并且没有进程打开这个文件，文件才会真正的被删除，并且内存空间才会被释放。如果删除文件是最后一个指向文件的链接，但是还有进程打开文件，文件会一直存在直到打开的文件描述符被关闭。可以使用unlink()创建临时文件，创建一个文件之后，使用unlink()删除文件，但是当前还有文件描述符指向文件，所以文件还在内存中，直到文件描述符关闭文件才会被删除。

### utime

```
#include <sys/types.h>
#include <utime.h>

int utime(const char *filename, const struct utimbuf *times);
```

utime()是一个系统调用，用来更改文件的最后一次访问时间和修改时间。结构体utimbuf的定义如下：
```
struct utimbuf {
   time_t actime;       /* access time */
   time_t modtime;      /* modification time */
};
```
actime为访问时间，modtime为修改时间。

### 目录的创建和销毁

```
#include <sys/stat.h>
#include <sys/types.h>

int mkdir(const char *pathname, mode_t mode);
```

mkdir()函数的作用是创建一个目录。

```
#include <unistd.h>

int rmdir(const char *pathname);
```
rmdir()函数的作用是删除一个目录，且这个目录必须是空的。

### 更改当前工作路径

```
#include <unistd.h>

int chdir(const char *path);
```

chdir()函数的作用是修改当前进程的工作目录。

### 分析目录/读取目录内容

```
#include <glob.h>

int glob(const char *pattern, int flags,
                int (*errfunc) (const char *epath, int eerrno),
                glob_t *pglob);
```

glob()函数的作用是寻找所有匹配通配符的文件路径。glob()参数解析：

- pattern: 通配符。
- flags: 是一个标记为，用来定义的glob()函数行为，可以通过按位或设置多个行为。flags取值参考man手册。
- errfunc: 是一个指向函数的指针，用于处理出错的路径和出错的原因。
- pglob: 为一个指向glob_t结构体的指向，用来存储glob()函数的执行结果。glob_t结构体的定义结构如下：
  ```
   typedef struct {
               size_t   gl_pathc;    /* Count of paths matched so far  */
               char   **gl_pathv;    /* List of matched pathnames.  */
               size_t   gl_offs;     /* Slots to reserve in gl_pathv.  */
           } glob_t;
  ```

函数执行成功返回0。其他情况的返回值：
- GLOB_NOSPACE: 内存空间不足.
- GLOB_ABORTED: 读取错误。
- GLOB_NOMATCH: 未找到匹配的路径。

## 系统数据文件和信息

