---
layout: post
title: "Shared libraries简介"
date: 2014-12-08 19:55:01 +0800
comments: true
categories: 
- linux
- ld
---

[pipe-1]: /images/assets/pipe-1.png "linux"

本篇主要说明进程间通信，也就是IPC（interProcess Communication）。UNIX系统IPC是各种进程通信方式的统称。这里主要讲PIPE和FIFO两种进程间通信的方式。


# PIPE

PIPE是UNIX一开始就支持的进程间通信的方式，有以下限制：

* 半双工的，通道的一端，要么写，要么读，不能支持读和写两个操作（虽然现代系统有的支持全双工，但是不可移植）
* 管道是字节流，这意味着没有消息块或者消息边界的概念。可以读或者写任意大小的数据。
* 使用PIPE进行进程通信的条件是“只能在具有公共祖先的进程之间使用”
* PIPE的容量是有限的。PIPE的实现实际上就是内核在内核空间开辟了一块内存，因此是有容量限制的。一旦空间满了，后续写入的就会被阻塞。


## 创建和使用管道

使用这个函数创建一个pipe。

```c
#include <unistd.h>
int pipe(int filedes[2]);
```

这个函数会通过参数filedes返回两个文件描述符。一个filedes[0]用来读，一个filedes[1]用来写。

因为是文件描述符，因此可以使用read()和write()系统调用来完成正常的I/O操作。

为了实现进程间通信，需要在调用pipe函数之前，调用fork来创建子进程，图示：

![alt text][pipe-1]

典型例子：

```c
int filedes[2];
if (pipe(filedes) == -1)
    errExit("pipe");
switch (fork()) {
case -1:
    errExit("fork");
case 0:  /* Child */
    if (close(filedes[1]) == -1)
        errExit("close");
    /* Child now reads from pipe */
    break;
default: /* Parent */
    if (close(filedes[0]) == -1)
        errExit("close");
    /* Parent now writes to pipe */
break; }
```

管道在不使用的时候需要关掉，这对正确使用管道来说是非常必要的。

当管道的一端关闭后，下列两条规则起作用：

* 当读一个写端已被关闭的管道时，在所有数据被读取后，read返回0，表示到达文件末尾。
* 如果写一个读端已关闭的管道，则产生信号SIGPIPE，如果忽略或者捕捉改信号并从其处理程序返回，则write返回-1，errno设置为EPIPE。

## 与shell打交道的popen()函数


常见的操作是创建一个管道连接到另一个进程，然后读其输出或向其输入端发送数据，为此，标准IO库提供了两个函数popen和pclose

```c

#include <stdio.h>
FIFE *popen(const char *cmdstring, const char *type);
int pclose(FIFE *fp);
```

# FIFO

FIFO被称为命名管道，PIPE只能是具有共同祖先的进程之间的通信。通过FIFO，不相关进程也可以通信。

FIFO是一种文件类型，可以通过stat结构成员st_mode来指示是否是FIFO类型，可以用宏S_ISFIFO对此进行测试。

创建一个FIFO：

```c
#include <sys/stat.h>

int mkfifo(const char *pathname, mode_t mode);

```

参数mode与open函数的mode相同。

一旦创建了FIFO文件，就可以用open打开，一般的文件I/O函数（close、write、read、unlink）都可用。

当打开一个FIFO时，非阻塞标志（O_NONBLOCK）产生下列影响：

* 如果没有指定O_NONBLOCK，只读open要阻塞到某个其它进程为写而打开此FIFO。类似，只写open要阻塞到某个其他进程为读而打开。
* 如果指定了O_NONBLOCK，则只读open立即返回。但是如果没有进程已经为读而打开一个FIFO，则open将出错并返回-1，error是ENXIO。

FIFO有两种用途：

* FIFO由shell命令使用，以便将数据从一条管道线传送到另一条，为此无需创建中间临时文件
* FIFO用于客户进程-服务器进程应用程序中。

FIFO可以被用于复制串行管道命令之间的输出流，于是也就不需要写数据到中间磁盘文件。管道只能用于进程间的*线性连接*。因为FIFO具有名字，因此可以用于非线性连接。

比如：

				|------------->prog3
				|
输入文件-----> prog1
				|
				|------------->prog2

实现这种过程，而不需要使用临时文件，可以使用FIFO以及tee命令（tee程序将其标准输入同时复制到其标准暑促以及其命令行中包含的文件名中）。

mkfifo	fifo1
prog3 < fifo1 &
prog1 < infile | tee fifo1 | prog2


# XSI IPC





































