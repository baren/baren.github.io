---
layout: post
title: "pthread control"
date: 2014-09-17 21:19:03 +0800
comments: true
categories: 
- linux
- apue
---

[linux]: /images/assets/pthread-1.png "linux"
[psd]: /images/assets/pthread-psd.png "linux"
[psd-key]: /images/assets/pthread-psd-key.png "linux"
[psd-key-1]: /images/assets/pthread-psd-key-2.png "linux"



# 线程属性

在使用pthread_create函数：

```c
int pthread_create(pthread_t *thread, const pthread_attr_t *attr, void *(*start)(void *), void *arg);
```

创建线程时，第三个参数attr是线程属性。可以使用pthread_attr_t结构来修改线程默认属性。

pthread_attr_t变量需要初始化，需要使用pthread_attr_init函数进行初始化。调用初始化函数后，pthread_attr_t结构所包含的内容就是操作系统实现支持的线程所有属性的默认值。
<!-- more -->
pthread_attr_t属性的初始化和销毁接口：

```c

#include <pthread.h>
int pthread_attr_init(ptread_attr_t *attr);
int pthread_attr_destory(pthread_attr_t *attr);

```

POSIX.1支持的线程属性包括：

* 线程的分离状态属性
* 线程栈末尾的警戒缓冲区大小
* 线程栈的最低地址
* 线程栈的大小

如果创建的线程不需要知道线程的终止状态，可以在创建的时候，以分离状态启动。通过设置pthread_attr_t的值为分离状态。设置pthread_attr_t的函数是：

```c
#include <pthread.h>
int pthread_attr_getdetachstate(pthread_attr_t *attr, int *detachstate);
int pthread_attr_setdetachstate(pthread_attr_t *attr, int detachstate);


```
设置分离状态的属性值是：PTHREAD_CREATE_DETACHED

# 同步属性

互斥量、读写锁和条件变量的属性

## 互斥量属性

主要讲互斥量属性的类型属性。类型属性有以下几种：

* PTHREAD_MUTEX_NORMAL——不检查死锁错误，如果一个线程试图去lock一个他已经锁住的互斥量，则发生死锁。
* PTHREAD_MUTEX_ERRORCHECK——提供错误检查
* PTHREAD_MUTEX_RECURSIVE——允许同一个线程多同一个互斥量多次加锁。会维持一个加锁计数量。

对互斥量，属性初始化和销毁函数是：

```c
#include <pthread.h>
int pthread_mutex_init(pthread_mutex_t *mutex, const pthread_mutexattr_t *attr);

int pthread_mutex_destroy(pthread_mutex_t *mutex);
```

可以通过下面函数设置互斥量属性：

```c
int pthread_mutexattr_gettype(pthread_mutexattr_t *attr, int *type);
int pthread_mutexattr_settype(pthread_mutexattr_t *attr, int type);


```


## 读写锁属性

支持进程共享属性

## 条件变量属性

支持进程共享属性

# 重入

在信号处理时，可重入的函数是指函数没有访问静态数据结构，或者没有调用malloc和free等。在线程处理时，函数同样也有重用的概念。

在多线程环境下，一个函数是可重入的，意思是：一个函数在同一时刻可以被多个线程安全调用。

注意与信号的区别
* 一个函数对多线程来说是可重入的，意思是这个函数是线程安全的。
* 但并不意味着对信号处理程序来说该函数是可重入的（比如标准io函数，是线程安全的，会对流加锁保证，但是对信号处理是不可重入的，因为会修改全局数据结构）。

有一个列表，列出了posix.1中不能保证线程安全的函数：

![alt text][linux]

如果操作系统实现线程安全这一特性时，会同时提供一个对应的线程安全版本。

比如asctime，对应的就是asctime_r，后缀是_r表示可重入。

posix.1还提供了以线程按方式管理FILE对象的方法。

标准IO流的实现，会对流加锁解锁操作，如果频繁调用getc函数，会有性能下降，因为会有频繁的加锁解锁。

为了解决这个问题，posix1.c引入非可重入版本的流函数：

```c

#include <stdio.h>
int getchar_unlocked(void); 
int getc_unlocked(FILE *fp);

putchar_unlocked(int c);
int putc_unlocked(int c, FILE *fp);

```

同时提供了线程安全的方式管理FILE对象的方法：

```c

#include <stdio.h>
int ftrylockfile(FILE *fp);
void flockfile(FILE *fp); 
void funlockfile(FILE *fp);

```

使用非线程安全的流函数版本时，需要用flockfile和funlockfile包围，否则会出现不可预测的问题（因为是非线程安全的）。

好处：

* 一旦对FILE对象进行加锁，就可以在释放锁之前对这些函数进行多次调用。这样就可以在多次数据读写上分摊总的加解锁开销。

# 线程私有数据

让一个函数是线程安全的最有效的方式就是让函数是可重入的。新的函数最好这么实现。尽管如此，
一些老的函数，不是线程安全的，如果要将其改造成线程安全函数，需要满足：

* 实现线程安全的
* 不能改变函数的签名，也就是不需要调用这个函数的程序去修改

使用线程私有（thread-specific）数据可以实现。

> 注意：
> 要理解线程私有数据，以函数的角度考虑问题。

线程私有数据允许函数为每一个线程维持一个单独的数据拷贝。如图所示：

![alt text][psd]

线程A调用函数myfunc时，myfunc函数为线程A维持一个单独数据，线程B调用myfunc函数时，myfunc为线程B维持一个线程B单独的数据。

线程私有数据有个特点：

* 存储的数据是持久化的，每一份数据会一直存在，这允许函数间共享数据（虽然不推荐）。

## 从函数角度考虑线程私有数据

为了更好的理解线程私有数据，需要从函数角度（实现角度）考虑如何使用线程私有数据

* 在线程第一次调用函数时，函数为线程分配独立的存储块。存储块只分配一次，就是在线程第一次调用此函数时。
* 同一个线程对这个函数的随后的调用，函数能够获取这个第一次调用而分配存储块。因此不能用局部变量存储指向存储块的key；也不能用static变量存储，因为在进程内，只有一个static的实例。
* 不同的函数可能都需要线程私有数据，因此每个函数都得需要自己的线程私有数据key
* 当线程停止时，函数不需要控制私有数据，因为停止时，代码有可能已经执行到函数外了。因此需要有个地方来执行清理操作。

## 线程私有数据 API

### 创建私有数据key

创建一个key，两个用处：

* 用来获取函数分配的存储块
* 用来区分其它函数的线程私有数据对应的key

使用pthread_key_create函数

```c
#include <pthread.h>
int pthread_key_create(pthread_key_t *key, void (*destructor)(void *));
```

destructor指向一个清理函数，用来释放函数内分配的存储块。

其签名是：

```c
void dest(void *value)
{
/* Release storage pointed to by 'value' */ 
}

```
线程停止时，并且这个key关联的数据不是NULL时，就会自动调用这个函数来清理。

一般，线程私有数据的实现，使用一个全局数组来存储这个key，这个key有两个状态：

* 是否使用的标记
* 清理函数指针

如图：

![alt text][psd-key]

根据图，pthread_key_create()返回的一般是全局数组的索引，数组元素包含两个字段，是否使用字段和清理函数地址字段。


### 关联函数分配内存与key

使用函数pthread_setspecific函数来关联函数分配的存储和pthread_key_create创建的key。

```c
#include <pthread.h>
int pthread_setspecific(pthread_key_t key, const void *value);
// value参数是一个分配的内存的指针
// 当线程停止时，这个值会传给create函数指定的清理函数。
void *pthread_getspecific(pthread_key_t key);


```

为了维护线程私有数据，Pthreads API 为每一个线程维护了一个指针数组，数据元素是函数分配的存储的指针。

如下图，假设pthread_keys[1]是函数myfunc分配的key，对于每一个线程，pthread api维护了一个指针数组，
数组元素指向函数内分配的内存，

![alt text][psd-key-1]


## 例子

### 非线程安全的

下面是一个非线程安全的strerror()的实现：


```c
#define _GNU_SOURCE  /* Get '_sys_nerr' and '_sys_errlist' declarations from <stdio.h> */
#include <stdio.h>
#include <string.h>   /* Get declaration of strerror() */
#define MAX_ERROR_LEN 256  /* Maximum length of string returned by strerror() */
static char buf[MAX_ERROR_LEN];  /* Statically allocated return buffer */
char *
strerror(int err)
{

	if (err < 0 || err >= _sys_nerr || _sys_errlist[err] == NULL) {
		snprintf(buf, MAX_ERROR_LEN, "Unknown error %d", err);
	} else {
		strncpy(buf, _sys_errlist[err], MAX_ERROR_LEN - 1);
		
		buf[MAX_ERROR_LEN - 1] = '\0'; /* Ensure null termination */
	}
	return buf; 
}


```

### 线程安全例子

```c

#define _GNU_SOURCE		/* Get '_sys_nerr' and '_sys_errlist' declarations from <stdio.h> */
#include <stdio.h>
#include <string.h>		/* Get declaration of strerror() */
#include <pthread.h>
#include "tlpi_hdr.h"


static pthread_once_t once = PTHREAD_ONCE_INIT; 
static pthread_key_t strerrorKey;


#define MAX_ERROR_LEN 256		/* Maximum length of string in per-thread buffer returned by strerror() */

static void
destructor(void *buf)		/* Free thread-specific data buffer */
{
	free(buf);
}

static void
createKey(void)		/* One-time key creation function */
{
	int s;
	/* Allocate a unique thread-specific data key and save the address of the destructor for thread-specific data buffers */

	s = pthread_key_create(&strerrorKey, destructor);
	if (s != 0)
		errExitEN(s, "pthread_key_create");
}

char *
strerror(int err)
{
    int s;
    char *buf;

	/* Make first caller allocate key for thread-specific data */

	s = pthread_once(&once, createKey); 
	if (s != 0)
		errExitEN(s, "pthread_once");

	buf = pthread_getspecific(strerrorKey);
	if (buf == NULL) { /* If first call from this thread, allocate buffer for thread, and save its location */

 		buf = malloc(MAX_ERROR_LEN);
        if (buf == NULL)
            errExit("malloc");

		s = pthread_setspecific(strerrorKey, buf); 
		if (s != 0)
			errExitEN(s, "pthread_setspecific");
	}
	
	if (err < 0 || err >= _sys_nerr || _sys_errlist[err] == NULL) { 
		snprintf(buf, MAX_ERROR_LEN, "Unknown error %d", err);
	} else {
		strncpy(buf, _sys_errlist[err], MAX_ERROR_LEN - 1);
		buf[MAX_ERROR_LEN - 1] = '\0'; /* Ensure null termination */
	}
	
	return buf; 
}
```

# 取消选项










