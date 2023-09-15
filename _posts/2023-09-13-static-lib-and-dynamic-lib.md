---
layout: post
title: 静态库和动态库的实现
categories: [Linux]
description: 静态库和动态库的实现
keywords: Linux, C
---

## 静态库和动态库的实现

在C语言开发中，我们会用到到一些外部实现的库，在使用前先要添加头文件。比如说我需要使用一些输入输出的函数，首先需要添加`#include <stdio.h>`语句。包含了头文件，就是对头文件中包含的函数进行了声明和定义。为什么包含了头文件之后，就能找对应的函数？是因为相关库已经存放在特定的路径下，比如`/usr/include`：
```
1$ ls /usr/include/
X11          cpio.h      fcntl.h            getopt.h        libintl.h   mtd        nfs             re_comp.h    signal.h       sysexits.h   ulimit.h
aio.h        crypt.h     features-time64.h  glob.h          limits.h    net        nl_types.h      regex.h      sound          syslog.h     unistd.h
aliases.h    ctype.h     features.h         gnu-versions.h  link.h      netash     nss.h           regexp.h     spawn.h        tar.h        utime.h
alloca.h     dirent.h    fenv.h             gnumake.h       linux       netatalk   obstack.h       resolv.h     stab.h         termio.h     utmp.h
ar.h         dlfcn.h     finclude           grp.h           locale.h    netax25    paths.h         rpc          stdc-predef.h  termios.h    utmpx.h
argp.h       drm         fmtmsg.h           gshadow.h       malloc.h    netdb.h    poll.h          rpcsvc       stdint.h       tgmath.h     values.h
argz.h       elf.h       fnmatch.h          iconv.h         math.h      neteconet  printf.h        sched.h      stdio.h        thread_db.h  video
arpa         endian.h    fstab.h            ifaddrs.h       mcheck.h    netinet    proc_service.h  scsi         stdio_ext.h    threads.h    wait.h
asm-generic  envz.h      fts.h              inttypes.h      memory.h    netipx     protocols       search.h     stdlib.h       time.h       wchar.h
assert.h     err.h       ftw.h              iproute2        misc        netiucv    pthread.h       semaphore.h  string.h       tirpc        wctype.h
byteswap.h   errno.h     gawkapi.h          langinfo.h      mntent.h    netpacket  pty.h           setjmp.h     strings.h      ttyent.h     wordexp.h
c++          error.h     gconv.h            lastlog.h       monetary.h  netrom     pwd.h           sgtty.h      sudo_plugin.h  uchar.h      x86_64-linux-gnu
complex.h    execinfo.h  gdb                libgen.h        mqueue.h    netrose    rdma            shadow.h     syscall.h      ucontext.h   xen
```

另外还可以通过环境变量设置路径。

库分为动态库和静态库，动态库在标准的位置，大家都可以使用，静态库可以不在标准的位置。静态库在编译的时候就会装载进来，在调用的时候不占用调用时间，大量使用静态库会造成程序体积膨胀。动态库在声明的时候只是引用动态库，在执行时候才回去动态库所在位置检查动态是否，并进行调用，会占用运行时间不占用编译时间。

### 静态库

静态库的名字一般为libxx.a，xx为库名。创建静态库命令:
```
ar -cr libxx.a yyy.o
```

然后发布到：
```
#存放头文件
/usr/local/include
#存放库文件
/usr/local/lib
```

使用静态库的方法：
```
gcc -I/usr/local/include -L/usr/local/lib -o ... -lxx
```
如果库发布到/usr/local/lib下面了，-L可以省略。-l参数必须在最后，为依赖的库名。



### 动态库

动态库的名称为libxx.so.n，xx为动态库的名称，n为版本号。

通过一下命令查看依赖的动态库：
```
ldd - print shared library dependencies
```

生成动态库命令：
```
gcc -shared -fpic -o libxx.so yyy.c
```
然后将生成的动态库和头文件发布到
```
/usr/local/include
/usr/local/lib
```
另外如果动态库的路径不在标准路径下，可以将路径添加到/etc/ld.so.conf中，然后使用命令/sbin/ldconfig重新读取/etc/ld.so.conf配置文件。

动态库的使用：
```
gcc -o main main.c -lxxx
```
如果有动态库和静态库同名，一定会有优先使用动态库，这是由内核决定的。如果依赖的库有多个，且依赖的库直接由依赖关系，被依赖的库应该放在后面，比如使用到libaa.so和libbb.so，libaa.so依赖于libb.so：
```
gcc -o main main.c -laa -lbb
```

除了标准库，使用到库必须使用-l包含。

如果以非root用户发布：
```
cp xxx.so ~/lib
export LD_LIBRARY_PATH=~/lib
```

使用静态库或动态库的时候，可以将代码中引入依赖的方式换成尖括号。
