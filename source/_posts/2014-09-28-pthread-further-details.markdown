---
layout: post
title: "pthread 进一步细节"
date: 2014-09-28 20:43:21 +0800
comments: true
categories: 
- linux
- pthread
---

# 线程和信号

信号的设计实现早于线程，导致线程与信号之间有许多冲突。冲突的地方主要在于：

* 既要维护传统的在单线程进程环境下的信号语义
* 同时又要开发符合多线程进程环境下的信号程序

<!-- more -->

## 信号模式怎样映射到线程

需要弄清楚信号的哪些方面是进程级别的，哪些方面是针对特定线程的。说明：

* 信号动作是*进程*级别的。这意味着一个默认动作是终止进程的信号被内核递送到进程的任意线程，所有的进程内的线程都会终止。
* 信号处理是*进程*级别的。进程内的所有线程都共享同一个信号处理器。如果一个线程使用sigaction()建立的信号处理器，进程内的任意线程如果收到这个信号，都会调用这个处理器；如果一个线程设置了忽略某个信号，其它线程也自动忽略
* 如果信号递送给一个多线程的进程时，内核会随机选取一个线程，用来递送信号和执行信号处理程序。
* 屏蔽信号是基于线程的。线程可以使用pthread_sigmask()来屏蔽某些信号。
* 内核维护了针对进程的一个全局的阻塞（pending）的信号集；也维护了针对每个线程的阻塞的信号集。sigpending()函数会返回进程的全局阻塞的信号集和调用这个函数的线程的阻塞信号集的并集。一个新创建的线程，阻塞的信号集是空。如果一个线程阻塞了一个信号，这个信号一直被阻塞，直到线程取消阻塞或者线程停止。
* 如果一个信号处理中断了pthread_mutex_lock()调用，这个调用总是会自动启动。如果中断了pthread_cond_wait()，也会自动重启。



以下几种情况，信号会递送给特定线程：

* 在某个线程上下文中，执行了一个特定的硬件指令的直接结果，生成信号
* 线程试图写入到一个终端的pipe文件时，生成一个SIGPIPE
* 使用pthread_kill() 或 pthread_sigqueue()可以允许同一进程的线程给另一个线程发信号


```c
#include <signal.h>
int pthread_kill(pthread_t thread, int sig);

int pthread_sigmask(int how, const sigset_t *set, sigset_t *oldset);


```

# 线程和进程控制

就像信号一样exec(), fork(), and exit()也是早于线程。

##### 线程和exec()

当任意线程调用exec()函数时，调用的程序会被完全取代。所有的线程，除了调用线程外，都会被终止。不会去执行线程的终止处理函数，属于进程的锁和条件变量都会消失。

##### 线程和fork()

多线程环境的进程调用fork()函数，只有调用进程在子进程中复制。其它线程在子进程中会消失，线程特定的析构或者清理函数都不会执行。这会导致严重问题：

* 虽然只有调用线程被复制了，但是进程全局状态，还有pthread对象（比如锁和条件变量）等全局对象，都会被复制到子进程。这会导致，如果在fork的同时，有个线程锁住了mutex，并且部分更新了全局数据结构，这种情况下，子进程中的线程将不能够解锁这个mutex。更进一步，子进程中的全局状态可能处于不一致状态。

* 由于线程的析构或者清理函数没有被调用，有可能导致子进程的内存泄露。

由于在多线程环境下调用fork有这么多严重问题，通常的建议是：

* 只有在后续立即调用exec函数的情况下，才会调用fork函数。否则不要调用。

如果必须调用fork函数，但是后面不跟着调用exec函数，linux提供了一个这种情况下的解决办法：

```c
pthread_atfork(prepare_func, parent_func, child_func);


```


# 线程实现模式

有许多线程实现方式，主要的区别是线程和内核的调度实体（kernel scheduling entities）之间映射的不同。

内核调度实体（kernel scheduling entities）是内核分配CPU和其他资源的单元。在传统的UNIX系统下，内核调用实体与进程是同义的。

## Many-to-one (M:1) 实现，用户级别线程

在M:1线程实现模式下，所有的线程创建、调度和同步细节都在用户空间下的线程lib包实现。内核不知道关于线程的任何细节。

这种实现有以下几种好处：

* 最大的好处是线程操作非常快速，因为这些全部在用户空间实现，不用切换到内核空间。
* 移植性好，由于是在用户空间实现的，可以很容易从一个系统移植到另一个系统

不好处是：

* 当一个线程调用了一个系统调用（比如read()），控制从用户空间的线程包转到内核，这意味着read()被阻塞了，那么其它线程就全部被阻塞了。
* 内核不能够调度线程。由于内核不知道线程的存在，因此内核不能调度线程到其它cpu上。

## One-to-one (1:1)实现方式

在1:1实现方式下，一个线程对应一个内核的调度实体。内核单独的处理每个线程的调度。这样，就解决了M:1的重大调度问题。

但是这种实现方式下，也有其它问题：

* 线程的创建同步等操作就比较慢，因为需要切换到内核空间下。
* 一对一的关系，内核需要为每一个线程维护一个内核调度实体，如果有大量的线程，会降低整体性能。

尽管如此，1:1的方式是大多数pthread线程实现的方式。两个linux的pthread实现都是采用1:1的方式

## linux的pthread实现方式

linux有两种pthread的实现：

* LinuxThreads: 这是linux的最初实现。Xavier Leroy开发
* NPTL (Native POSIX Threads Library): 新的linux下的pthread实现。Ulrich Dreppe（gun c也叫libc的管理者） 和 Ingo Molnar实现。性能比LinuxThreads好，也更符合pthread的规范。

在glibc 2.4及其后续版本，不再支持LinuxThreads了。

### LinuxThreads实现细节

* 使用clone() 系统调用创建一个线程，复制的资源是

CLONE_VM | CLONE_FILES | CLONE_FS | CLONE_SIGHAND

这意味着LinuxThreads线程共享虚拟内存、文件描述符、文件相关信息（umask, root directory, and current working directory）和信号处理。

* 除了由应用创建的线程，LinuxThreads还创建了额外的管理线程，来处理线程的创建的销毁。

* 实现采用了信号来进行内部的操作。

当内核支持实时信号（Linux 2.2及其以后），前三个实时信号被使用；如果是老的内核，使用 SIGUSR1 and SIGUSR2，这样，应用不能够使用这几个信号。

关于实时信号，参考：
http://unixhelp.ed.ac.uk/CGI/man-cgi?signal+7

### NPTL

* 使用clone() 创建线程，复制的资源是

CLONE_VM | CLONE_FILES | CLONE_FS | CLONE_SIGHAND | CLONE_THREAD | CLONE_SETTLS | CLONE_PARENT_SETTID | CLONE_CHILD_CLEARTID | CLONE_SYSVSEM

* 使用了前两个实时信号

* 不像LinuxThreads，NPTL没有实现管理线程





