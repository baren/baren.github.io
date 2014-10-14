---
layout: post
title: "取消线程"
date: 2014-09-25 20:56:04 +0800
comments: true
categories: 
- linux
- pthread
---

主要描述POSIX线程的取消机制和更进一步的线程细节，包括线程和信号，线程栈等。

<!-- more -->

# 取消一个线程

使用pthread_cancel函数取消特定的线程：

```c

#include <pthread.h>
int pthread_cancel(pthread_t thread);
// 返回0表示成功

```

pthread_cancel只是发送取消请求，然后立即返回。这意味着调用线程不用等待线程停止。目标线程什么时候停止，取决于目标线程的状态和类型。

# 取消状态和类型


使用pthread_setcancelstate()设置线程的取消状态；使用pthread_setcanceltype()设置线程的取消类型。这两个状态设置线程如何响应取消操作的。

pthread_setcancelstate()函数可设置的状态是：

* PTHREAD_CANCEL_DISABLE。 线程是不可取消的。这种线程如果接收到一个取消请求，会保持未决（pending）状态直到成为可取消状态
* PTHREAD_CANCEL_ENABLE。线程是可取消的，这个状态也是默认的状态。

线程在执行一段必须执行完的代码时，设置为不可取消状态，是非常有用的。

pthread_setcanceltype()函数可以设置两种类型：

* PTHREAD_CANCEL_DEFERRED。线程一直执行直到遇到取消点（特殊函数）。默认类型。
* PTHREAD_CANCEL_ASYNCHRONOUS。线程可以在任意时间点取消，一般不大永。

# 取消点

当一个线程是可取消的并且是延迟的（PTHREAD_CANCEL_ENABLE和PTHREAD_CANCEL_DEFERRED）。取消操作会在线程执行到下一个取消点时起作用。

SUSv3 定义了一组必须是取消点的函数，还定义了一组是可选取消点的函数。

可取消函数列表（略）

对于一个不是分离的线程，必须由其它函数调用pthread_join函数等待这个线程结束。如果这个线程接收了取消请求，并到达了一个取消点，则pthread_join返回的值是PTHREAD_CANCELED.

取消线程的例子：

```c

#include <pthread.h>
#include "tlpi_hdr.h"

static void *
threadFunc(void *arg)
{
	int j;
	printf("New thread started\n");
    for (j = 1; ; j++) {
        printf("Loop %d\n", j);
        sleep(1);
    /* NOTREACHED */
    return NULL;
}

int
main(int argc, char *argv[])
{
    pthread_t thr;
    int s;
    void *res;

    s = pthread_create(&thr, NULL, threadFunc, NULL); 
    if (s != 0)
		errExitEN(s, "pthread_create"); 

	sleep(3);		/* Allow new thread to run a while */

	s = pthread_cancel(thr);
	if (s != 0)
		errExitEN(s, "pthread_cancel");

	s = pthread_join(thr, &res);
	if (s != 0)
		errExitEN(s, "pthread_join");

	if (res == PTHREAD_CANCELED) 
		printf("Thread was canceled\n");

	else
		printf("Thread was not canceled (should not happen!)\n");
    exit(EXIT_SUCCESS);
}
```

# 测试取消点

如果线程没有调用这些取消点函数（纯计算线程），为了也能够响应取消请求，可以使用pthread_testcancel()来当取消点。

```c
#include <pthread.h>
void pthread_testcancel(void);
```

# 清理处理器

如果一个线程接收到取消请求，执行到一个取消点，则会停止。有可能会导致共享的变量和pthread对象（比如锁）处在不一致状态，可能会导致剩下的线程死锁等异常状态。

为了避免这个问题，需要定义线程结束的清理函数。

每一个线程都有一个线程处理函数栈。当线程被取消时，从上到下依次开始执行清理处理程序。

```c

#include <pthread.h>
void pthread_cleanup_push(void (*routine)(void*), void *arg);
void pthread_cleanup_pop(int execute);

```

> 注意：
> 当线程正常执行完成，不会触发清理处理函数。

一般来说，一个清理操作只有在执行一段特殊的代码时被取消时，才会用到。


下面例子在主main函数中创建了一个线程，他分配了一块内存，并锁住了一个互斥锁mtx。因为线程有可能被取消，因此使用pthread_cleanup_push()来安装清理处理函数，这个清理函数主要作用是释放分配的内存，并对互斥锁解锁。

安装完清理处理器后，线程进入所谓的特殊代码段（如果取消，需要走清理处理函数的）。

如果特殊代码段正常执行完成，则调用pthread_cleanup_pop()去掉处理函数。

例子：

```c

#include <pthread.h>
#include "tlpi_hdr.h"

static pthread_cond_t cond = PTHREAD_COND_INITIALIZER;
static pthread_mutex_t mtx = PTHREAD_MUTEX_INITIALIZER;
static int glob = 0; /* Predicate variable */

static void 		/* Free memory pointed to by 'arg' and unlock mutex */ cleanupHandler(void *arg)
{
	int s;

	printf("cleanup: freeing block at %p\n", arg);
	free(arg);

	printf("cleanup: unlocking mutex\n");
	s = pthread_mutex_unlock(&mtx);
	if (s != 0)
		errExitEN(s, "pthread_mutex_unlock");
}


static void *
threadFunc(void *arg)
{
    int s;
    void *buf = NULL;	/* Buffer allocated by thread */
	
	buf = malloc(0x10000);
	printf("thread: allocated memory at %p\n", buf);
	
	s = pthread_mutex_lock(&mtx); 	/* Not a cancellation point */ 
	if (s != 0)
		errExitEN(s, "pthread_mutex_lock");

	pthread_cleanup_push(cleanupHandler, buf);
	
	while (glob == 0) {
		s = pthread_cond_wait(&cond, &mtx); 	/* A cancellation point */
		if (s != 0)		
			errExitEN(s, "pthread_cond_wait"); 
	}
	printf("thread: condition wait loop completed\n");
    pthread_cleanup_pop(1);
    return NULL;
}

int
main(int argc, char *argv[])
{
    pthread_t thr;
    void *res;
    int s;

    s = pthread_create(&thr, NULL, threadFunc, NULL); 
    if (s != 0)
		errExitEN(s, "pthread_create");

	sleep(2);		/* Give thread a chance to get started */

	if (argc == 1) {		/* Cancel thread */
	    printf("main: about to cancel thread\n");
    	s = pthread_cancel(thr);
    	if (s != 0)
			errExitEN(s, "pthread_cancel");
	} else {		/* Signal condition variable */
		printf("main: about to signal condition variable\n");
		glob = 1;
		s = pthread_cond_signal(&cond);
		if (s != 0)
			errExitEN(s, "pthread_cond_signal");
	}

	s= pthread_join(thr, &res); 
	if (s != 0)
		errExitEN(s, "pthread_join"); 

	if (res == PTHREAD_CANCELED)
		printf("main: thread was canceled\n"); 
	else
		printf("main: thread terminated normally\n");
    
    exit(EXIT_SUCCESS);
}	
```

> 注意：
> 注意上面例子对pthread_cleanup_push()函数的的典型使用








