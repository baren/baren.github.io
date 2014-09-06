---
layout: post
title: "apue(进程控制)"
date: 2014-05-04 20:00:02 +0800
comments: true
categories: linux
---


[file]: /images/assets/Figure8-1.png "time-function"
[fork]: /images/assets/Figure8-2.png "time-function"

# 进程标识符

进程都有一个非负整数代表唯一的进程ID。

当进程终止后，进程ID可以重用，为了防止将新进程视为使用同一个进程ID的旧的进程，系统采用了*延迟重用*算法来重用进程ID。

系统启动后有一些专用进程：

* ID为0的进程——调度进程，也称为交换进程（swapper），这个进程是内核的一部分，不执行磁盘上的程序，被称为系统进程。
* ID为1的进程——init进程，自举结束时由内核调用；早期的进程文件是/etc/init，现在都是/sbin/init。init通常读取系统的配置文件来初始化系统（/etc/rc*或/etc/initab, /etc/init.d）。init进程不会终止，虽然是普通进程，但是以超级用户权限运行，还接收孤儿进程，是所有孤儿进程（子进程还活着，父进程终止了，称为孤儿进程）父进程。

下面函数返回与进程相关的其他标识符：

```

#include <unistd.h>

pid_t getpid(void);

pid_t getppid(void);

uid_t getuid(void);  // 实际用户id

uid_t geteuid(void);  // 有效用户id

gid_t getgid(void);  // 实际组id

gid_t getegid(void);  // 有效组id

```

<!-- more -->

# fork函数

可以调用```fork```函数创建一个新进程。

```

#include <unistd.h>

pid_t fork(void);

```

fork创建的新进程被称为子进程。fork调用一次返回两次：

* 子进程返回0.
* 父进程返回子进程的ID，如果返回-1，表示创建进程出错

返回后，父进程和子进程继续执行调用fork之后的指令。

fork后，子进程是父进程的副本。比如子进程获得父进程的数据空间，堆和栈的副本。父子进程不共享这些，只是共享正文段。

一般fork后都执行exec函数，因此现在的系统在fork的实现上，并不立即完全复制父进程的数据段、堆栈的完全复制，而是采用copy-on-write的技术，只有在父或子进程写的时候才进行复制，复制也是仅仅复制写的那块区域，一般就是存储系统中的一页。

在fork后，是先返回父进程还是子进程，是不确定的，取决于系统的调度算法的实现。

如果fork后没有调用exec，子进程与父进程执行同一代码，则需要注意调用fork之前的IO流缓冲问题。

* 如果之前有IO流缓冲，则可能会被子进程和父进程各调用一次，伪代码：

```

printf("。。。");  // 如果标准输出被重定向到一个文件，则是全缓冲的。

pid = fork()
if(pid == 0){
	exit()
} else if (pid == -1) {
	printf('error')
} else {
	exit()
}

```

上面代码，如果printf的标准输出重定向到文件，则是全缓冲的，因此调用printf的时候，有可能把数据存放到缓冲中。

这样，fork后，父子进程各有一个缓冲，因此exit后，printf各被flush，因此父子进程都会打印printf的输出。

> 注意：
> strlen() 和sizeof()的区别。strlen是返回不包含null终止符的字符串的长度。sizeof返回包含null终止符的缓冲区的大小。
> strlen() 是一次函数调用。而sizeof在编译时期就知道大小。

## 文件共享

fork的一个特征是文件进程所有打开文件的描述符都被复制到子进程。

这样父子进程每个相同的打开描述符共享一个文件表项（内核为每个打开的文件维持文件表，若两个进程打开同一个文件，每个进程都有自己的文件表项。但是fork后，两个进程就会共享同一个文件表项，也就是共享偏移量。）

如图：


如果两个进程打开同一个文件，每个进程都有自己的文件偏移量：
![alt text][file]

如果fork后，父进程和子进程则共享一个文件表项，共享文件偏移量：
![alt text][fork]


fork后处理文件描述符的两种情况：

1. 父进程等子进程完成，在这种情况下，父进程无需对描述符进行任何处理，当子进程结束后，父进程的偏移量也已经被更新。
2. 父子进程各自执行不同的程序段，fork，父子进程各自关闭不需要的文件描述符，这样不会干扰对方使用的文件描述符。网络服务进程常用这种方式。

除了文件，许多其它属性也被子进程继承：

* 实际用户（组）ID，有效用户（组）ID
* 会话ID
* 环境


注意：
 父进程的文件锁不会被继承

 fork的两种用法：

 1. 父进程希望复制自己，是父子进程执行不同的代码段（比如网络服务）
 2. 子进程执行一个不同的程序，比如shell

# vfork函数

vfork的目的是创建一个新进程，而该进程的目的是exec一个新进程。

vfork与fork调用相同，有两点不同：

* vfork不会复制父进程的地址空间，在子进程调用exec或exit之前，子进程在父进程的地址空间中执行（意味着会修改父进程的变量等）
* vfork函数保证子进程先执行，在它调用exec或exit之后，父进程才被调度运行

# exit函数

5种正常终止进程的方式：

* main函数中return返回，等效于调用exit函数
* 调用exit函数。ISO C定义的函数，操作包括1）调用终止处理程序（atexit注册的函数）；2）关闭所有标准IO流描述符
* 调用_exit或_Exit函数。ISO C定义_Exit函数。这两个函数的目的是提供一种无需运行终止处理程序（atexit注册的函数）或信号处理程序而终止的方法；这两个函数一般都不对标准IO流进行冲洗。
* 进程最后一个线程在启动例程中执行返回语句，进程终止状态为0
* 进程最后一个线程调用pthread_exit函数，进程终止状态为0

3种异常终止程序：

* 调用abort
* 进程收到某些信号，这个信号可以由其它进程、自己或内核发起。
* 最后一个线程对“取消”做出响应。

> 注意：
> 不管是正常终止或者是异常终止，内核都会在进程结束后执行一段代码，工作是：
> 1. 关闭打开的描述符
> 2. 释放使用的存储器

## 子进程如何通知父进程它的终止状态

1. 对于三个终止函数（exit, _exit, _Exit）退出的进程，若通知父进程其退出状态，需要把状态作为参数传递给这三个函数。其它两种正常退出，终止状态为0.
2. 异常终止，内核产生一个指示其异常终止原因的终止状态。

> 注意：
> 不管哪种情况，父进程都可以通过wait和waitpit函数获取其终止状态。

## 进程领养与僵死进程

父子进行结束的两种情况：

1. 父进程先于子进程终止。

对于父进程已经终止的子进程，其父进程改变为init进程（ID为1）。每当一个进程终止时，内核会挨个检查所有活动的进程，判断是否是已终止进程的子进程。是，则将其父进程修改为init进程。

这个步骤称为*进程的领养*。

2. 子进程先于父进程终止。

为了让父进程获取其子进程的终止状态，内核为每个终止的子进程保存了一定量的信息。因此当父进程调用wait或waitpid函数时，可以获取子进程的终止状态。

内核为终止的子进程保存的信息有：

* 进程ID
* 终止状态
* 进程使用CPU时间总量

> 注意：
> 由于进程终止时，内核释放了其占用的存储空间和关闭了打开的文件描述符，引起，进程占用的大部分资源都已经释放掉了。

*僵死进程（zombie）* ：一个已终止，而父进程没有终止，并且父进程还没有对其进行善后处理（获取子进程的终止信息，也就是还没有经过wait调用）的进程，被称为僵死进程，状态为Z。

> 注意：
> 由init进程领养的子进程终止时，不会称为僵死进程。因为init的实现，只要有子进程终止，就会调用wait函数获取其终止状态。

# 父进程获取子进程终止状态（wait和waitpid函数）

当进程的子进程终止时，内核会立即向其父进程发送SIGCHLD信号。因此父进程可以提供一个信号处理程序处理这个信号。

对于这个信号，父进程可以有两种处理方式：

* 编写一个处理这个信号的处理程序（然后调用wait函数获取其终止子进程的终止状态）
* 忽略掉（系统默认）

除了信号方式外，若父进程在程序内调用wait或者waitpid函数，则可能会发生以下：

* 若所有子进程都在运行，阻塞
* 若一个子进程已经终止，正等待父进程获取其终止状态，则会立刻返回
* 若没有任何子进程，立即出错返回

```
#include <sys/wait.h>

pid_t wait(int *statloc);

pid_t waitpid(pid_t pid, int *statloc, int options);

```

wait和waitpid函数的区别：

* 在一个子进程终止前，wait函数会使其调用至阻塞，而waitpid有一个选项，可使调用者不阻塞（比如：```waitpid(-1, &statloc, WNOHANG)```）
* waitpid可以不用等待其调用后的第一个终止的子进程，可以指定它等待的进程（参数pid > 0时）

参数statloc是一个整型指针，子进程的终止状态存放于此，若不关心终止状态，可以传空指针。而判断子进程的终止状态，可以使用wait.h定义的宏：

宏        | 描述          	
 ------------- | :-------------: 
WIFEXITED		| 若正常终止的子进程，则为真。此时，可以调用WEXITSTATUS(status)获取子进程传送给exit _exit _Exit参数的低8位
WIFSIGNALED		| 若为异常终止的子进程返回的状态，则为真。可以使用WTERMSIG(status)获取子进程终止的信号编号。
WIFSTOPPED 		| 若为当前暂停子进程的返回状态，则为真。可使用WSTOPSIG(status)获取进程暂停信号编号
WIFCONTINUED	| 若作业控制暂停后已经继续的子进程返回了状态，则为真。


## waitpid

waitpid函数功能多于wait函数，其中，pid参数的意义：

* pid == -1 ，等待任一子进程，此时与wait等效
* pid > 0，等待进程Id等于PID的进程
* pid == 0，等待其组ID等于调用进程组ID的任意一子进程
* pid < -1，等待其组ID等于pid绝对值的任一子进程


而waitpid的options参数进一步控制了waitpid的操作：

* WNOHANG : 不阻塞
* WCONTINUED : 作业控制相关 
* WUNTEACED: 作业控制相关

依据这两个参数，waitpid提供了wait不具有的三个功能：

* 可等待特定的进程
* 提供了wait的非阻塞版本
* 支持作业控制

# 关于僵死进程补充

## 僵死进程的危害


若父进程一直存在，但不调用wait回收终止子进程的终止状态，则：

* 内核保留了僵死进程的一定量信息，资源浪费
* 进程号一直没有释放，也就得不到重用

## 避免僵死进程的方法

* 通过信号机制
* wait/waitpid函数调用
* fork两次，由init来回收

## 如何查看僵死进程

命令

```
ps -A -o stat, ppid, pid, cmd | grep -e '^[zZ]'

* -A 列出所有进程
* -o 自定义输出字段，stat表示进程状态，Z/z表示僵死进程

# 竞争条件

当多个进程都企图对共享数据进行某种处理，而最后的结果又取决于进程运行的顺序时，则认为发生了*竞争条件（race condition）*


```

# exec函数

在fork后，如果调用exec函数，则该进程执行的程序完全替换成新程序，并且从新程序的main函数开始执行。exec并不创建新进程，因此调用exec前后并不改变进程ID。exec只是用全新的程序替换了当前进程的正文数据堆和栈等段。


unix系统提供了6个exec函数，unix对进程的控制：

* fork
* exit
* wait
* exec

这几个函数使得unix进程控制原语更加完善。


```
#include <unistd.h>

// 下面四个函数，执行程序是路径
int execl(const char *pathname, const char *arg0, ... /* (char *)0 */)  // 参数是列表，以空指针结束

int execv(const char *pathname, char *const argv[]) // 参数列表是一个数组

int execle(const char *pathname, const char *arg0, ... /* (char *)0, char *const envp[] */) // 可以传递一个环境表，这个环境表是个指针数组，每个元素是指向字符的指针

int execve(const char *pathname, char *const argv[], char *const envp[]) // 传递环境表


// 执行程序是文件名，如果文件名以“/”开头，则认为是路径，否则从PATH变量中搜寻执行文件
int execlp(const char *filename, const char *arg0, ... /* (char *)0 */)

int execvp(const char *filename, char *const argv[])


```

这六个函数的几种区别是：

1. 前四个函数已路径名作为参数，而后两个使用文件名作为参数，如果是文件名作为参数，则：
	* 如果filename是包含“/”，则认为是路径
	* 否则就按PATH环境变量，在它所指定的各目录中搜寻可执行文件（如果找到的可执行文件不是链接器生成的机器可执行文件，则认为是shell脚本，就会调用/bin/sh，并以filename作为shell的输入）

> 说明：
> 环境表：每个程序都会接收一张环境表，这个环境表是一个字符指针数组，全局变量environ指向这个数组。
>		环境表包含的环境变量，比如PATH=:/bin:/usr/bin:、USER=sar、等
>
> 环境变量（环境表中的每个数组元素），可以通过getenv获取环境变量。通过putenv、setenv等设置环境变量，一般使用这两个
> 函数访问设置环境变量，而不推荐直接访问environ全局变量。
>
> POSIX.1 XSI等预定义了很多环境变量，其中PATH环境变量表示“搜索可执行文件的路径前缀列表”


2. 第二个区别是参数表的传递不同。l表示list，v表示vector。execl、execle、execlp的每个命令行参数都说明为一个单独的参数，最后以空指针结尾。而execv、execve、execvp的命令行参数是以数组提供。

3. 最后一个区别，传递的环境表有关。已e结尾的两个函数execle和execve，可以传递一个指向环境字符串指针数组的指针，其它几个函数则使用调用进程中的environ变量为新程序复制现有的环境。


注意几点：

1. 对打开的文件的处理

这与执行时关闭close-on-exec设置有关

进程中每个打开描述符都有一个执行时关闭标志。若设置此标志，则在执行exec时关闭该描述符，否则该描述符仍然打开。除非特地用fcntl设置了该标志，否则系统默认的操作是在执行exec后仍保持这种描述符的打开。

> fcntl函数可以：
> * 复制一个现有描述符
> * 设置/获得文件描述符标记（文件描述符标志，当前只定义了一个，就是FD_CLOEXEC）
> * 获得/设置文件状态标志
> * 等。

posix.1明确要求在执行exec时关闭打开的目录流，这通常是opendir来实现的。它调用fcntl函数为对应于打开目录流的描述符设置执行时关闭标志。

> 疑问：
> 设置close-on-exec的场景是什么？？

2. 有效用户问题

exec后，实际用户id和实际组id不变，而有效id是否改变，则取决于程序文件的设置用户id和设置组id是否设置。

很多unix实现中，只有execve是系统调用，其它函数是库函数。

# 更改用户ID和组ID

unix系统中，特权是基于用户和组ID的。

可以调用setuid函数设置实际用户id和有效用户id。setgid函数设置实际组id和有效组id。

```
#include <unistd.h>

int setuid(uid_t uid);

int setgid(gid_t gid);

```

对于修改ID有以下规则：

* 若进程具有超级用户特权，则setuid函数将实际用户ID、有效用户ID以及保存的设置用户ID设置为参数uid
* 若进程没有超级用户特权，但是参数uid等于实际用户ID**或者**保存的设置用户ID，则setuid只将有效用户ID设置为uid，不改变实际用户ID**和**保存的设置用户ID
* 若上面两个条件都不满足，则将errno设置为EPERM，并返回-1.

关于内核维护的三个用户ID，注意：

1. 只有超级用户进程可以更改实际用户ID。通常实际用户ID是在用户登录时，有login程序设置的，而且拥有不会改变他。
2. 仅当对程序文件设置了设置用户ID位时，exec函数才会设置有效用户ID。如果设置用户位没有设置，则exec函数不会改变有效用户ID，而将其维持为原先的值。任何时候都可以调用setuid，将有效用户ID设置为实际用户ID。
3. 保存的设置用户id是由exec复制有效用户ID而得来的。如果设置了文件的设置用户ID位（执行位显示为s），则exec根据文件的用户ID设置了进程的有效用户ID以后，就将这个副本保存起来。


ID    | exec(设置用户ID位关闭) | exec（设置用户ID位开启）| setuid（超级用户） | setuid（非特权用户）          	
 ------------- | :-------------: | :-------------: | :-------------: | :-------------: 
 实际用户ID 	| 不变	| 不变	| 设置为uid	|不变
 有效用户id 	| 不变	| 设置为程序文件的用户ID	| 设置为uid	|设置为uid
 保存的设置用户ID| 从有效用户ID复制	| 从有效用户ID复制|设置为uid | 不变

## su 和sudo和setuid的关系

su命令可以切换用户，而sudo可以以超级用户权限运行命令。查看这两个文件状态：

```
$ ll /usr/bin/su
-rwsr-xr-x  1 root  wheel  21472  3 13  2013 /usr/bin/su
$ ll /usr/bin/sudo
-r-s--x--x  1 root  wheel  164496  3 13  2013 /usr/bin/sudo

```

su和sudo的用户执行位都是s，表示setuid为开启。当用户运行这个程序时，

会设置程序的有效用户ID为文件所有者，就会以这个文件的所有者（这里是root）权限运行。

因此，sudo命令可以以root权限执行命令。

当su命令执行时，设置进程的有效用户ID为root。

# 解释器文件

解释器文件，是文本文件，起始行的形式是：

```
#! pathname [optional-argument]

```

感叹号和pathname之间的空格是可选的。最常见的解释器是：

```
#! /bin/sh
```

* pathname通常是绝对路径，对它不进行特殊处理（既不视屏PATH进行路径搜索），对这种文件识别，是由内核作为exec系统调用处理的一部分完成的。
* 内核调用exec函数实际上执行的并不是解释器文件，而是解释器文件中第一行pathname指定的文件
* 注意区别解释器文件（文本文件，以#!开头）和解释器之间区别（解释器文件第一行pathname指定）

## 解释器例子

```
#include <unistd.h>
#include <stdio.h>
#include <sys/wait.h>

int main(int argc, char *argv[])
{
	pid_t pid;
	if((pid = fork()) == -1)
	{
		printf("fork error\n");
		exit(-1);
	}
	if(pid == 0)
	{
		printf("child process\n");
		if(execl("/Users/user/work/cproj/testinterp", "testinterp", "myarg1", "My arg2", (char *) 0) < 0)
		{
			printf("error execl\n");
			exit(-1);
		}
		exit(0);
	}
	if(waitpid(pid, NULL, 0) < 0)
	{
		printf("error waitpid");
		exit(-1);
	}
	exit(0);

}


```

解释器文件testinterp很简单，只有一行：

```

#! /Users/user/work/cproj/echoarg foo
```

而echoarg程序仅仅是打印参数：

```
#include <stdlib.h>
#include <stdio.h>

int main(int argc, char *argv[])
{
	int i;
	for (i = 0; i < argc; i++)
	{
		printf("ARGV[%d]:%s\n", i, *(argv+i));
	}
	exit(0);
}

```

结果如下：

```
ser@usertekiMacBook-Pro cproj$ ./execinterp
child process
ARGV[0]:/Users/user/work/cproj/echoarg
ARGV[1]:foo
ARGV[2]:/Users/user/work/cproj/testinterp
ARGV[3]:myarg1
ARGV[4]:My arg2

```

对程序结果的解释：

* 当exec调用解释器（/Users/user/work/cproj/echoarg）时，argv[0]是该解释器的pathname（）。
* argv[1]是解释器文件中的可选参数（foo）
* 其余参数是pathname（/Users/user/work/cproj/testinterp）
* 以及第二、三个参数（myarg1和My arg2）
* 注意，第一个参数（testinterp），是没有用的

## 解释器文件可选参数例子

解释器pathname后可跟随可选参数，比如awk支持-f选项：

```

awk -f myfile
```

它告诉awk从文件myfile中读awk程序。

如果在解释器文件中使用-f选项，则解释器文件可以这么写：

```

#! /bin/awk -f

后面跟着awk程序
```

比如解释器文件/usr/local/bin/awkexample的内容是：

```
#!/bin/awk -f 
BEGIN {
	for (i = 0; i < ARGC; i++)
		printf "ARGV[%d] = %s\n", i, ARGV[i]
	exit 
}

```
调用：

```
$ awkexample file1 FILENAME2 f3 ARGV[0] = awk
ARGV[1] = file1
ARGV[2] = FILENAME2
ARGV[3] = f3
```

如果执行这个文件时，实际上，是这么调用的：

```
/bin/awk -f /usr/local/bin/awkexample file1 FILENAME2 f3
```

1. 解释器文件的路径名/usr/local/bin/awkexample给传给解释器，因为不知道解释器是不是会从搜索路径中搜索该文件，因此传全路径文件名
2. awk读取解释器文件，第一行是#，注释，因此忽略第一行，执行后续程序



