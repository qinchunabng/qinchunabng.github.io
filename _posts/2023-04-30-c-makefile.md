---
layout: post
title: C语言中的makefile
categories: [C]
description: C语言中的makefile
keywords: C, makefile
---

makefile是用来管理工程的脚本文件。在一些大的工程中，会分为多个模块，各个模块之间相互依赖，makefile用来定义模块的依赖关系，方便编译。

加入现在有个工程，工程中文件如下：
```
-- main.c
|- tool1.h
|- tool1.c
|- tool2.h
|- tool2.c
```
main.c依赖tool1.c和tool2.c，需要将整个工程编译为一个可执行文件，makefile定义如下：
```
mytool:main.o tool1.o tool2.o
    gcc main.o tool1.o tool2.o -o mytool

main.o:main.c
    gcc main.c -c -Wall -g -o main.o

tool1.o:tool1.c
    gcc tool1.c -c -Wall -g -o tool1.o

tool2.o:tool2.c
    gcc tool2.c -c -Wall -g -o tool2.o
```

上面脚本的含有为：编译目标为mytool的可执行文件，mytool是通过main.o,tool1.o,tool2.o编译而成的，下面再列出mytool的编译命令。下面列出了main.o,too1l.o,tool2.o文件如何编译。相当于是从叶子到根的一棵树，先去产生各个`.o`文件，再生产可执行文件。编译时在makefile所在文件执行make命令即可。在编译一次之后再执行编译，编译器会对比可执行文件和依赖的`.o`文件的时间戳，如果可执行文件和`.o`都是最新的说明可执行文件时最新的，不需要重新编译，否则会重新编译。

一般可以再makefile中定义clean指令：
```
clean:
    rm *.o mytool -rf
```
通过执行`make clean`，会清理生成的可执行文件和所有的`.o`文件。

在makefile中，可以通过定义变量的方式，来简化makefile的写法，比如上面的makefile可以简化为：
```
OBJS:main.o tool1.o tool2.o
CC=gcc
CFLAGS+=-c -Wall -g

mytool:$(OBJS)
    $(CC) :$(OBJS) -o mytool

main.o:main.c
    $(CC) main.c $(CFLAGS) main.o

tool1.o:tool1.c
    $(CC) tool1.c $(CFLAGS)  tool1.o

tool2.o:tool2.c
    $(CC) tool2.c  $(CFLAGS) tool2.o

clean:
    $(RM) *.o mytool -rf
```
上面makefile中，定义了OBJS,CC,CFLAGS三个变量，OBJS为编译的对象，CC为编译器（默认为gcc）,CFLAGS为编译选项，引用变量通过`$(变量名)`的方式引用。另外，在下一行中引用上一行中相同的变量，可以通过`$^`引用，还可以通过`$@`引用上一句中的目标文件。并且main.o,tool1.o,tool2.o的编译是一样的，所以可以通过通配符模板代替。所以上面makefile也可以写为：
```
OBJS:main.o tool1.o tool2.o
CC=gcc
CFLAGS+=-c -Wall -g

mytool:$(OBJS)
    $(CC) :$(OBJS) -o $@

%.o:%.c
    $(CC) main.c $(CFLAGS) $@

clean:
    $(RM) *.o mytool -rf
```