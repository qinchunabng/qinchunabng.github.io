---
layout: post
title: 进程的概念
categories: [Linux]
description: 进程的概念
keywords: Linux, C
---

### 进程标识符pid

进程描述符的类型为pid_t，为16位有符号整型，也就是同时能产生3万多个进程。进程号是顺次向下使用的，通过getpid()获取进程id，通过getppid()获取分进程的id。函数原型：
```
pid_t getpid(void);
pid_t getppid(void);
```

### 进程产生

进程通过fork()和vfork函数产生.
```
pid_t fork(void);
```
fork()函数的作用是创建复制当前调用进程来产生一个新的进程，新创建的进程称为子进程，当前调用进程称为父进程。父进程和子进程拥有不同的内存空间，在fork()函数调用的那一刻，两个进程的内存中内容是一样的。但是一个进程的写操作和文件映射操作不会影响另外一个进程。-fork后父子进程的区别：
- fork的返回值不一样
- pid不同
- ppid不一样
- 未决信号和文件锁不继承
- 资源利用量清零
  
init进程是所有进程的祖先进程，进程号是1。

fork()调用成功的话，在父进程中返回子进程的PID，在子进程中返回0。调用失败父进程返回-1，子进程不会被创建，errno会被设置为对应的错误码。

fork()示例：
```
#include <stdio.h>
#include <stdlib.h>
#include <sys/types.h>
#include <unistd.h>

int main(){
    pid_t pid;

    printf("[%d]:Begin\n",getpid());

    pid = fork();
    if(pid < 0){
        perror("fork()");
        exit(1);
    }
    
    if(pid == 0){
        printf("[%d]:Child is working!\n",getpid());
    }else{
        printf("[%d]:Parent is working!\n", getpid());
    }

    printf("[%d]:End!\n",getpid());
    exit(0);
}
```
输出结果：
```
[75]:Begin
[75]:Parent is working!
[75]:End!
[76]:Child is working!
[76]:End!
```
子进程创建之后会执行`pid==0`分支的逻辑，然后在向下执行打印输出End。父进程会执行else的逻辑，然后打印End。可以用`ps -axf`查看父子进程之间的关系。

如果我们用重定向将输出结果输出到一个文件中:`./fork > /tmp/out`，查看传输文件的内容：
```
[88]:Begin
[88]:Parent is working!
[88]:End!
[88]:Begin
[89]:Child is working!
[89]:End!
```
这是由于在fork子进程的之前没有刷新缓存区，会将父进程的输出缓冲区的内存复制到子进程，所以子进程会再输出一次Begin。但是为什么在终端中没有这个问题，是因为终端是标准输出，标准输出都是行缓存，加换行符就会刷新缓存区，但是文件流是全缓存，换行符只代表换行。所以要解决这个问题，可以在fork()函数之前加上`fflush(NULL)`刷新缓冲区。

### 进程的消亡和释放资源

父进程创建的子进程需要父进程来回收释放，否则子进程会称为僵尸进程。比如一个进程中创建了N个子进程来执行任务，子进程任务执行完之后父进程没有回收，子进程就会变为僵尸进程，只到父进程执行结束，子进程变为孤儿进程被Init进程接管回收。

另外还存在一个旧版vfork()函数，也是用来创建子进程的，它与fork()函数的区别在于：fork函数会把父进程的内存完完整整拷贝一份，给进程使用，而vfork()函数创建的子进程不会这么做，创建的子进程指向物理内存与父进程是同一块内存。

如果来回收子进程，下面看一个函数：
```
pid_t wait(int *wstatus);
pid_t waitpid(pid_t pid, int *wstatus, int options);
```
这些系统调用是用来获取在子进程状态发生变化时获取子进程的信息。变化的状态为：子进程终止，子进程被一个信号停止，或者子进程被一个信号恢复。如果子进程终止，执行wait操作可以让系统释放子进程的资源；如果没有执行wait操作，终止的子进程会保持"zombine"状态。如果子进程的状态发生变化，wait()函数会直接返回，否则会一直阻塞直到子进程状态发生变化或者一个信号中断了这个系统调用。

wait()系统调用会阻塞当前调用线程，直到任何一个当前进程的子进程终止。这个系统调用等同于：
```
waitpid(-1, &wstatus, 0);
```

waitpid()系统调用的作用是阻塞调用线程直到pid指定的子进程状态发生变化。默认情况下，waitpid()只等待终止的子进程，但是通过options参数修改这个行为。pid的参数取值如下：
- < -1。意味等待进程group ID为pid绝对值的任何子进程。
- -1。等待任何子进程。
- 0。等待进程group ID为当前调用进程group ID的子进程。
- \> 0。等待进程ID为pid的子进程。

options为以下取值，可以为以下一个或多个常量值，通过或操作设置多个值：
- WNOHANG。如果没有子进程退出直接返回。
- WUNTRACED。如果子进程（没有通过ptrace追踪）停止则返回。即使没有设置这个参数，被追踪的进程停止了状态也能获取到。
- WCONTINUED。如果停止的进程会SIGCONT信号恢复函数会返回。

如果传入的`wstatus`参数不为NULL，会将状态填充到`wstatus`指针所指向的内存。状态值可以用系统提供的一些宏来判断，下面列出一些场景的宏（注意下面宏传入的整型值，不是指针）：
-  WIFEXITED(wstatus)：返回true，表示子进程正常终止，也就是通过调用exit(3)或者_exit(2)函数，或者从main函数返回。
-  WEXITSTATUS(wstatus)：返回子进程的退出状态，但是这个宏只能在WIFEXITED(wstatus)返回为true的状态下才能调用。
-  WIFSIGNALED(wstatus)：如果子进程被一个信号终止返回为true。
-  WTERMSIG(wstatus)：如果WIFSIGNALED(wstatus)返回为true，用这个宏获取导致子进程终止的宏的编号。
-  WCOREDUMP(wstatus)：如果子进程产生了core dump文件返回true。这个宏应该在WIFSIGNALED(wstatus)返回true的时候调用。

### exec函数族

```
 extern char **environ;

int execl(const char *pathname, const char *arg, ...
                       /* (char  *) NULL */);
int execlp(const char *file, const char *arg, ...
                       /* (char  *) NULL */);
int execle(const char *pathname, const char *arg, ...
                       /*, (char *) NULL, char *const envp[] */);
int execv(const char *pathname, char *const argv[]);
int execvp(const char *file, char *const argv[]);
int execvpe(const char *file, char *const argv[],
                       char *const envp[]);
```
exec函数族的作用是将当前进程的镜像替换为一个新的进程镜像。这些函数的初始化的参数是要执行文件的名称。这写函数可以按照exec后的字母进行分类：
- l - execl(),execlp(),execle()
  
  arg参数以及后面的`...`可以看作arg0,arg1...argn，它们是执行程序所需要的参数，最后以后必须以(char *)NULL作为终结符结尾，因为这是一个可变参数。第一个参数为要执行的文件的名称。

- v - execv(),execvp(),execvpe()
  
  `char *const argv[]`是一个以NULL结尾的字符串数组，这些参数新的进程可以为访问。第一个参数是要执行的文件的文件名。

- e - execle(),execvpe()

  envp参数定义执行进程的环境变量。envp参数是一个以NULL结尾的的字符串数组。其他所有的不以'e'结尾的exec()函数，环境变量都是通过调用进程的external变量environ获取的。 

- p - execlp(),execvp(),execvpe()

  这些函数可以实现shell通过可执行文件名搜索可执行文件的操作，如果文件名中不包过'/'。文件的查找路径通过PATH环境变量定义以冒号分隔，如果环境变量没有定义，默认路径包含了confstr(_CS_PATH)函数返回的目录（一般返回值为"/bin:/usr/bin")），或者当前工作目录。如果定义的文件名中包含了/，则不会从PATH中查找。

如果exec()函数返回了，说明发生了异常。如果返回值为-1，errno会设置为对应的错误编码。

示例：
```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>

int main(){
    puts("Begin!");
    fflush(NULL);

    execl("/bin/date", "date", "+%s", NULL);
    perror("execl()");
    exit(1);

    puts("End!");
    exit(1);
}
```
输出结果：
```
Begin!
1691384204
```
需要注意的是在执行execl()需要执行刷新缓冲区的命令。如果不加，由于puts()是行缓冲模式，但是在文件输出流中，'\n'刷新行缓冲会失效。例如，如果不加fflush(NULL)，执行`./execl > /tmp/out`，/tmp/out中只有`1691384204`的输出结果，不会有'Begin!'，因为在execl执行之后会替换当前进程的镜像，进程pid不变。一般情况下，exec()是与fork()配合使用的：
```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>

int main(){
    puts("Begin!");
    pid_t pid;

    pid = fork();
    if(pid < 0){
        perror("fork()");
        exit(1);
    }
    if(pid == 0){
        execl("/bin/date", "date", "+%s", NULL);
        perror("execl()");
        exit(1);
    }

    wait(NULL);

    puts("End!");
    exit(1);
}
```
输出结果：
```
Begin!
1691384960
End!
```

shell中执行命令的原理，就是fork一个子进程，然后用exec()函数替换子进程，shell中调用wait()等待回收子进程。

exec()函数argv0参数还有个作用是隐藏真实的进程名称，例如修改上面示例代码，调用sleep进程休眠：
```
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <sys/types.h>
#include <sys/wait.h>

int main(){
    puts("Begin!");
    pid_t pid;

    pid = fork();
    if(pid < 0){
        perror("fork()");
        exit(1);
    }
    if(pid == 0){
        execl("/bin/sleep", "httpd", "100", NULL);
        perror("execl()");
        exit(1);
    }

    wait(NULL);

    puts("End!");
    exit(1);
}
```
编译执行后，重新打开一个shell窗口执行`ps axf`:
```
 PID TTY      STAT   TIME COMMAND
    1 ?        Sl     0:00 /init
    7 ?        Ss     0:00 /init
    8 ?        S      0:00  \_ /init
    9 pts/0    Ss     0:00      \_ -bash
   81 pts/0    S+     0:00          \_ ./exec
   82 pts/0    S+     0:00              \_ httpd 100
   49 ?        Ss     0:00 /init
   50 ?        S      0:00  \_ /init
   51 pts/1    Ss     0:00      \_ -bash
   83 pts/1    R+     0:00          \_ ps axf
```
sleep进程被伪装成了一个httpd进程，一些木马程序会采用这样的方式进行伪装。

shell的命令分为内部命令和外部命令。区分是内部还是外部通过命令是否是存储在磁盘区分，存在磁盘上的为shell的外部命令，其他的为内部命令。

通过fork()和wait实现shell的例子：
```
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <glob.h>
#include <unistd.h>
#include <errno.h>

#define DELIMS " \t\n"

struct cmd_st
{
  glob_t globres;
};

static void prompt()
{
  printf("mysh-0.1$ ");
  fflush(NULL);
}

static void parse(char *line, struct cmd_st *cmd)
{
  int i = 0;
  int res;
  char *tok;
  while (1)
  {
    tok = strsep(&line, DELIMS);
    if (tok == NULL)
      break;
    if (tok[0] == '\0')
      continue;
    // it is not need to append in the first time.
    printf("tok:[%s]\n", tok);
    res = glob(tok, GLOB_NOCHECK | GLOB_APPEND * i, NULL, &cmd->globres);
    if (res != 0)
    {
      fprintf(stderr, "glob():%s\n", strerror(res));
      // return -2;
    }
    // printf("glob.gl_pathc:%ld\n", cmd->globres.gl_pathc);
    i = 1;
  }
  // return 0;
}

int main()
{
  char *linebuf = NULL;
  size_t linebuf_size = 0;
  struct cmd_st cmd;
  pid_t pid;
  int res;
  while (1)
  {
    prompt();
    if (getline(&linebuf, &linebuf_size, stdin) < 0)
    {
      perror("getline()");
      break;
    }
    // printf("linebuf:[%s]\n", linebuf);
    parse(linebuf, &cmd);
    // printf("%s:%d\n", __FUNCTION__, __LINE__);
    if (0)
    {
      //是内部命令
    }
    else
    {
      //是外部命令
      pid = fork();
      if (pid < 0)
      {
        perror("fork()");
        exit(1);
      }
      if (pid == 0)
      {
        // printf("gl_pathv[0]:%s\n", cmd.globres.gl_pathv[0]);
        execvp(cmd.globres.gl_pathv[0], cmd.globres.gl_pathv);
        perror("execvp()");
        exit(1);
      }
      else
      {
        wait(NULL);
      }
    }
  }
  exit(0);
}
```

### 用户组和权限

u+s权限就是一个可执行文件在执行的时候，权限切换为当前这个可执行文件的user的身份。执行shell命令的时候，流程是fork一个子进程然后再用exce执行shell命令，在shell进程中等待子进程执行结束回收。所以在执行u+s权限的执行文件的时候，切换用户权限过程是在exec函数中完成的，执行子进程的时候使用可执行文件user身份执行的。

Linux在启动之后会先以root的身份启动一个init进程，然后通过fork+exec启动一个getty进程，然后显示输入用户密码登录的提示，用户输入用户名后，通过exec将当前执行进程替换为login进程。在login进程执行时，会显示输入密码，输入密码之后，login进程会以root用户的身份读取shadow文件，判断用户名密码是否正确。如果验证正确，通过fork+exec启动一个shell进程，切换为当前用户的res，使用的是当前登录用户的身份。如果用户名密码验证失败，重新执行getty进程和login进程，输入用户名和密码。

```
uid_t getuid(void);
uid_t geteuid(void);
```

getuid()函数的作用是获取实际的用户ID，geteuid()函数的作用是获取有效的用户ID。

```
#include <sys/types.h>
#include <unistd.h>

int setuid(uid_t uid);
```

setuid()函数的作用是设置当前进程的有效用户ID。

结合上面的函数，实现一个类型登录的权限切换的操作：

```
#include <stdlib.h>
#include <stdio.h>
#include <unistd.h>
#include <sys/types.h>
#include <unistd.h>

int main(int argc, char **argv)
{
  pid_t pid;
  int ret;
  if (argc < 3)
  {
    fprintf(stderr, "Usage....\n");
    exit(1);
  }

  pid = fork();
  if (pid < 0)
  {
    perror("fork()");
    exit(1);
  }

  if (pid == 0)
  {
    // child process
    ret = setuid(atoi(argv[1]));
    if (ret < 0)
    {
      perror("setuid()");
      exit(1);
    }
    execvp(argv[2], argv + 2);
    perror("execvp()");
    exit(1);
  }
 
  wait(NULL);
  exit(0);
}
```

编译后，切换为root，`chown root:root ./mysu`修改可执行文件的mysu的用户为root，并且使用`chmod u+s ./mysu`给可以执行文件mysu添加u+s的权限。然后切换为普通用户，使用`./mysu 0 cat /etc/shadow`即可查看shadow文件的内容。

### 守护进程

守护进程就是一直在后台运行的进程，守护进程一般需要脱离控制终端，并且是一个会话的leader，还是一个进程组的leader，

```
#include <sys/types.h>
#include <unistd.h>

pid_t setsid(void);
```

setsid()函数的作用是如果调用进程不是一个进程组的leader就创建一个新的会话。当前进程会称为新会话的leader和新进程组的leader，并且脱离从控制终端。

```
#include <sys/types.h>
#include <unistd.h>

int setpgid(pid_t pid, pid_t pgid);
pid_t getpgid(pid_t pid);
```

setpgid()函数的作用是给进程id为pid的进程设置进程组ID。getpgid()函数的作用是获取进程的进程组ID。

### 系统日志

系统位于/var/log目录下，安装内容进行分类，messages为主日志文件。所有需要写系统日志进程将系统日志交给syslogd进程，再由syslogd进程将日志写入系统日志。

```
#include <syslog.h>

void openlog(const char *ident, int option, int facility);
void syslog(int priority, const char *format, ...);
void closelog(void);
```

openlog()函数的作用是与syslogd进程相关联，syslog()是提交日志内容到syslogd,closelog()是关闭与syslogd的关联。