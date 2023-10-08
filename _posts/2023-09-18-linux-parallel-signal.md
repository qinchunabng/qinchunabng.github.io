---
layout: post
title: Linux并发——信号
categories: [Linux]
description: Linux并发——信号
keywords: Linux, C
---

## 并发

同步就是执行执行的时候什么时候会执行什么代码是清楚的。异步就是什么事件时候到来不知道，会产生什么样的结果也不知道。

异步事件的处理：查询法和通知法。举个例子：一个人在河边钓鱼，鱼有没有上钩就是一个异步事件，鱼什么时候上钩是不知道。如果想知道有没有鱼上钩，过一会用抄网捞一下，看看有没有鱼上钩，这种方式就属于查询法。另外一份方式就是用鱼漂，有鱼上钩鱼漂会动，这就是通知法。

什么时候用通知法什么时候用查询法？如果当前异步事件发生的比较稀疏，用通知法比较合适。如果发生的频率比较高，使用查询发。

### 信号

信号是软件中断。信号的响应依赖于中断。

#### 信号的概念

通过`kill -l`命令可以查看所有的信号：
```
 1) SIGHUP       2) SIGINT       3) SIGQUIT      4) SIGILL       5) SIGTRAP
 6) SIGABRT      7) SIGBUS       8) SIGFPE       9) SIGKILL     10) SIGUSR1
11) SIGSEGV     12) SIGUSR2     13) SIGPIPE     14) SIGALRM     15) SIGTERM
16) SIGSTKFLT   17) SIGCHLD     18) SIGCONT     19) SIGSTOP     20) SIGTSTP
21) SIGTTIN     22) SIGTTOU     23) SIGURG      24) SIGXCPU     25) SIGXFSZ
26) SIGVTALRM   27) SIGPROF     28) SIGWINCH    29) SIGIO       30) SIGPWR
31) SIGSYS      34) SIGRTMIN    35) SIGRTMIN+1  36) SIGRTMIN+2  37) SIGRTMIN+3
38) SIGRTMIN+4  39) SIGRTMIN+5  40) SIGRTMIN+6  41) SIGRTMIN+7  42) SIGRTMIN+8
43) SIGRTMIN+9  44) SIGRTMIN+10 45) SIGRTMIN+11 46) SIGRTMIN+12 47) SIGRTMIN+13
48) SIGRTMIN+14 49) SIGRTMIN+15 50) SIGRTMAX-14 51) SIGRTMAX-13 52) SIGRTMAX-12
53) SIGRTMAX-11 54) SIGRTMAX-10 55) SIGRTMAX-9  56) SIGRTMAX-8  57) SIGRTMAX-7
58) SIGRTMAX-6  59) SIGRTMAX-5  60) SIGRTMAX-4  61) SIGRTMAX-3  62) SIGRTMAX-2
63) SIGRTMAX-1  64) SIGRTMAX
```
前面的数字代表的是信号值，SIGRTMIN到SIGRTMAX之间信号是实时信号。每个信号对应的都有默认动作，标准信号中大多数信号的默认动作都是终止+产生core文件，core文件包含程序出错的现场信息。

可以通过`ulimit -c`命令设置core文件的大小，这样如果程序中出现段错误，就会产生core文件。如果程序的时候加入-g调试选项，可以通过gdb查询core文件，分析错误产生的原因。

#### signal()

```
#include <signal.h>
typedef void (*sighandler_t)(int);
sighandler_t signal(int signum, sighandler_t handler);
```

signal()函数的作用是定义信号的行为。signum为信号值，handler是信号的回调函数。

**信号会打断阻塞的系统调用。**

#### 信号的不可靠

信号会打断阻塞的调用，返回EINTR的错误。

#### 可重入函数

什么是信号的可重入，就是信号的第一次调入还没有结束，就发送第二次调用。所有的系统调用都是可重入的，一部分库函数也是可重入的，比如说memcpy。如果一个函数提供了_r版本，那么这个函数的原函数一定不能出现再信号处理中。

#### 信号的响应过程

内核为每个进程维护两个位图，一个叫mask，一个叫pending。假如现在有个程序，功能是打印10个`*`，每隔1秒打印一个。程序中有一个main函数和一个handler函数，handler为信号处理函数。在main函数每隔一秒打印一个`*`，当按`ctrl+c`的时候，在信号处理函数handler中打印一个`!`。mask是信号屏蔽字，用来表示信号的状态，pending用来表示当前进程收到了哪些信号。mask位图的值一般情况下都是1，pending的值默认为0。信号从收到到响应有一个不可避免的延迟。

当进程因为时间片耗尽，保存执行现场后，从用户态切换到内核态，当前进程进入到一个等待队列，等待CPU的调度。直到当前进程重新获取CPU恢复执行，恢复执行时切换到用户态，通过`mask&pending`位运算来判断是否有信号，为0则无信号发生。当有信号发生时，会将pending中对应的位置为1，恢复调度的时候`mask&pending`运算就得到对应为1，说明有信号发生。信号发生时只时将pending中的位置为1，只有进程被中断打断，恢复执行时，才会检查中断位，才能发现是否有信号发生。当有信号发生时，先将对应的mask位和pending位置为0，将当前进程执行地址替换注册信号回调函数的地址，执行回调函数。信号处理函数处理完之后，切换为内核态，将进程的执行地址替换为原来的地址，将mask位恢复1。然后再从内核态变为内核态，做`mask&pending`的位运算，由于刚刚信号回调已经处理，进程的执行地址已经替换为最开始的执行地址，当前进程恢复执行中断前的代码。

标准信号的响应没有严格的顺序。

当设置信号处理方式为忽略时，就是将对应信号的mask位置为0，这样对应的信号就会被忽略。

#### 信号的常用函数

- kill()

  ```
  #include <sys/types.h>
  #include <signal.h>

  int kill(pid_t pid, int sig);
  ```

  kill()函数的作用是给指定的进程发送信号，信号值为sig。
  - 如果pid的值为正数，就是给进程号为pid的进程发送信号；
  - 如果pid的值为0，则给当前调用进程同组的进程发送信号，也叫组内广播。
  - 如果pid的值为-1，则给所有当前进程有权限发送信号的进程发送信号，除了init进程；
  - 如果pid小于-1，则给进程组ID为-pid的进程组内的进程发送信号。
  - 如果sig为0，没有信号会发送，但是权限检查和进程ID检查仍然会被执行，可以用来检查进程ID是否存在或者是否有权限给对应的进程组发送信号。
  
  成功返回0，失败返回-1，并且errno设置为对应的错误值。
  - EINVAL:一个无效信号值。
  - EPERM:调用进程没有权限给目标进程发送信号。
  - ESRCH:目标进程或进程组不存在。需要注意的是目标进程可能是一个僵尸进程，即可能已经终止执行，但是没有被父进程wait()回收。

- raise()

  ```
  #include <signal.h>

  int raise(int sig);
  ```
  raise()函数的作用是给调用进程或线程发送信号，在一个单线程的程序中等同于：
  ```
  kill(getpid(),sig);
  ```
  在一个多线程的程序中等同于：
  ```
  pthread_kill(pthread_self(),sig);
  ```
  如果信号触发了一个信号处理函数handler的调用，raise()在handler()函数返回后才会返回。

- alarm()

  ```
  #include <unistd.h>

  unsigned alarm(unsigned seconds);
  ```
  alarm()函数的作用是让系统在seconds秒之后，生成一个SIGALRM的信号（SIGALRM信号的默认动作是杀掉当前进程）。处理器调度延迟可能导致信号产生后信号处理函数不会立即被调用。

  如果seconds为0，表示取消alarm请求。

  如果设置了多个alarm()，只有一个会生效，后面会覆盖前面的alarm()请求。

  alarm()一般配合pause()函数使用。有部分平台的sleep()函数就是用alarm()+pause()实现的，如果在程序中使用了sleep()函数，并且程序还使用alarm()函数，可能会导致alarm()被覆盖，所以考虑到移植在程序中最好不好使用sleep()函数。

- pause()

  ```
  #include <unistd.h>

  int pause(void);
  ```
  pause()函数的作用是使调用进程（或线程）休眠，直到信号导致进程终止或者信号处理函数被调用。

- setitimer()

  ```
  #include <sys/time.h>
  int setitimer(int which, const struct itimerval *new_value,
                     struct itimerval *old_value);
  ```
  setitimer()用于更精确的实现定时控制。
  which为以下三种，用于不通的时钟计时方式和产生不同的信号：
  - ITIMER_REAL
    
    计时的方式为实时计时，到了时间之后产生一个SIGALRM信号。

  - ITIMER_VIRTUAL

    计时方式为进程在用户态所消耗的CPU时间（包含了进程中所有的线程所消耗的CPU时间），时间到后产生一个SIGVTALRM信号。

  - ITIMER_PROF

    计时方式为进程所消耗的总的CPU时间（user态和system态），时间到后产生一个SIGPROF信号。

  new_value用来定义定时器的到期时间，old_value如果不为NULL，将会指向timer的上一次设置的过期时间值。struct itimerval和struct timeval定义如下：
  ```
  struct itimerval {
    struct timeval it_interval; /* Interval for periodic timer */
    struct timeval it_value;    /* Time until next expiration */
  };

  struct timeval {
    time_t      tv_sec;         /* seconds */
    suseconds_t tv_usec;        /* microseconds */
  };
  ```
  struct itimerval有两个属性it_interval和it_value，计时递减的时it_value的值，到了时间之后it_value设置为it_interval（这个过程为原子的）。it_interval和it_value为struct timeval类型，struct timeval包含tv_sec和tv_usec两个属性，tv_sec为秒，tv_usec为毫秒。

- abort()

  ```
  #include <stdlib.h>

  void abort(void);
  ```
  abort()函数的作用时给当前进程发送一个SIGABRT信号，并且终止当前进程产生一个core.dump文件。

- system()

  ```
  #include <stdlib.h>

  int system(const char *command);
  ```
  system()函数使用fork()创建一个子进程，并且在子进程中执行command指定的shell命令，等同于：
  ```
  execl("/bin/sh", "sh", "-c", command, (char *) NULL);
  ```
  system()函数在command执行结束后返回。

  要保证system()函数正常执行，必须要阻塞SIGCHLD信号，忽略SIGINT和SIGQUIT信号。

- sleep()

  ```
  #include <unistd.h>

  unsigned int sleep(unsigned int seconds);
  ```
  sleep()函数会休眠当前线程seconds秒，或者一个未忽略的信号产生。sleep()可以用nanosleep()或usleep()替代。

#### 信号集

```
#include <signal.h>

int sigemptyset(sigset_t *set);

int sigfillset(sigset_t *set);

int sigaddset(sigset_t *set, int signum);

int sigdelset(sigset_t *set, int signum);

int sigismember(const sigset_t *set, int signum);
```

sigemptyset()作用是用来初始化一个信号集，信号集是以位图方式表示。

sigfillset()作用是填充信号集，包含所有的信号。

sigaddset()和sigdelset()作用分别是添加信号集和删除信号集。

sigismember()用来判断信号集中是否包含信号signum。

#### 信号屏蔽字&pending集的处理

```
#include <signal.h>

/* Prototype for the glibc wrapper function */
int sigprocmask(int how, const sigset_t *set, sigset_t *oldset);
```
sigprocmask()函数的作用是设置信号屏蔽字，来决定什么时候信号应该被处理。how用来定义行为，set用来定义操作的信号集，oldset用来保存旧的信号集。how的取值：
- SIG_BLOCK
  
  添加阻塞信号集（set定义的信号集和当前已经为阻塞的信号集），即将信号屏蔽字中对应信号的位置为0。

- SiG_UNBLOCK

  恢复被阻塞的信号集。

- SIG_SETMASK

  设置阻塞信号集为set。


```
#include <signal.h>

int sigpending(sigset_t *set);
```

sigpending()返回当前线程pending状态的信号集，表示收到的信号。

#### 扩展

- sigsuspend()

```
#include <signal.h>

int sigsuspend(const sigset_t *mask);
```

sigsuspend()作用是使用指定信号集零时替换当前线程的信号屏蔽字，阻塞当前进程（线程）等待某个信号或者进程终止。如果进程终止，sigsuspend()不会返回，如果信号被处理，信号处理函数返回后，sigsuspend()返回，并且恢复sigsuspend()函数执行前的信号屏蔽字，这整个过程是原子的。

- sigaction()

```
#include <signal.h>

int sigaction(int signum, const struct sigaction *act,
                     struct sigaction *oldact);
```

使用signal()函数时，如果多个信号使用同一个信号处理函数，由于信号允许存在中断嵌套，信号处理函数可能被同时调用多次。假如信号处理函数中有释放资源类似的操作，可能会导致内存泄漏。所以如果多个信号公用一个处理函数，在响应某个信号，需要阻塞其他信号，就需要使用sigaction()函数。另外sigaction()可以区分信号的来源。

- setitimer()

#### 实时信号

标准信号的响应没有严格意义上的顺序，实时信号是相对于标准信号而言的。实时信号是需要排队，响应是有顺序的。如果一个进程即受到标准信号又收到实时信号，首先响应标准信号。实时信号没有动作，实时信号是SIGRTMIN到SIGRTMAX之间的信号。

实时信号排队大小是有限制的，可以通过`ulimit -a`命令查看：
```
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
其中pending signals表示排队信号的大小，可以通过`ulimit -i`修改其大小。