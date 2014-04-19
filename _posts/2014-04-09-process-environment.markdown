---
comments: true
date: 2014-04-09 20:00:00
layout: post
title: apue(进程环境)
summary: 'apue chapter7'
tags:
- linux
- 笔记
---

[exit-func]: /assets/Figure7-1.png "time-function"
[process-mem]: /assets/Figure7-2.png "time-function"

# main函数

c程序的起始运行函数是

```
int main(int argc, char *argv[]);

```

argc是命令行参数数目，argv是指针数组，指向每一个参数。

内核执行C程序是调用exec函数，在调用main函数之前，会先调用一个特殊的启动例程，可执行程序文件指定这个例程作为程序的启动地址。这是由链接编辑器设置的。链接编辑器由编译器调用。

启动例程会从内核取得命令行参数和环境变量值。

# 进程终止

有八种终止进程的方式，其中5种方式为正常终止，为：

1. 从main函数返回
2. 调用exit函数
3. 调用_exit或_Exit
4. 最后一个线程从其启动例程返回
5. 最后一个线程调用pthread_exit

异常终止的三种方式：

6. 调用abort
7. 接到一个信号并终止
8. 最后一个线程对取消请求做出响应

程序启动例程，在main函数返回后，会立即调用exit函数。其类似过程是这样的：

```
exit(main(argc, argv));
```

实际上，并不是这样（过程是一样的），启动例程一般是用汇编编写。

## exit函数

三个用于正常终止程序的函数中，_exit和_Exit函数立即进入内核。

exit函数则先执行一些清理处理（调用终止处理程序，关闭所有标识I/O流），然后进入内核。

```

#include <stdlib.h>
void exit(int status);
void _Exit(int status);

#include <unistd.h>
void _exit(int status);

```

exit的清理操作，执行一个标准的I/O库的清理关闭操作，为所有打开流调用fclose函数（怎么实现的？）

> 注意
> exit和_Exit函数是ISO C标准定义的，而_exit则是POSIX说明的，因此使用的头不一样。

由于标准库有buffer，因此怎样退出程序很重要。

若使用fopen函数打开文件，并没用fclost关闭它，影响是什么呢？

若使用exit函数或正常从main函数返回，两种方式都会调用exit函数。因此，exit函数会为你处理：

* flush输出流
* 关闭文件
* 进程拥有的其它资源也会被释放

如果非正常退出，或调用_exit或_Exit函数，则

* 系统会关闭打开的文件
* 释放资源
* buffer并不会被flush

三个exit函数都有一个整型参数，称之为终止状态（或退出状态，exit status），shell可用$?查看上一条命令的返回值。

若：

* 调用这些函数不带终止符
* main执行了一个无返回值的return语句
* main没有生命返回类型为整型

则：

进程状态为未定义的。

但：

若main生命为int返回类型，并执行到最后一条语句时返回（隐式返回），

则：

进程终止状态为0

在main函数返回一整型值与用该值调用exit函数是等价的

```
exit(0) == return 0;
```

## atexit函数

可以注册终止处理函数，这批函数在exit自动调用，ISO C规定，可注册函数多达32个，注册这些函数使用atexit函数

```
#include <stdlib.h>

int atexit(void (*func)(void));

```

> 注：
> 1. exit调用这些函数顺序与他们登记的顺序相近
> 2. 同一个函数若登记多次，则会调用多次

![alt text][exit-func]

# 命令行参数

比较简单，记录一点。

argv[argc]是一个空指针，这样，可以在循环获取参数时，这样：

```

for(i=0; garv[i] != NULL; i++)
{
	// 代码
}
```
# 环境表

每个程序，都会收到一张环境表，环境表是字符指针数组，每个指针指向的字符以null结束。

**全局变量**```environ```包含了该指针数组的地址：

```
extern char ** environ;

```

称environ为环境指针，指针数组为环境表，环境表每个表项由name=value组成。

# C程序的存储空间

C程序由以下几部分组成

* 正文段：是由CPU执行的机器指令部分，正文段是共享的，而且是只读的。
* 初始化数据段：常称为数据段，它包含了程序中须明确的赋初始值的变量。比如
在C程序中出现在任何函数之外的声明：

```
int maxcount = 99;

```

* 非初始化数据段：常称为bss段，在程序开始执行之前，内核将此段中的数据初始化为0或空指针，出现在任何函数外的C声明：

```
long sum[1000];

```

* 栈：自动变量，函数调用时所需要保存的信息，都存放到栈中，这种实现方式允许递归调用。
* 堆：一般在堆中进行动态存储分配。堆位于非初始化数据段和栈之间。

示意图：

![][process-mem]


linux的size命令可以报告正文段、数据段和bss段的长度。

# 共享库

共享库使得可执行文件中不再需要包含共用库的例程。

只需要在所有进程都可引用的存储区中维护这种库例程的一个副本，在例程**第一次**执行或在第一次调用某个库函数时，用**动态链接**方法，将程序与共享库函数相链接。

好处：

* 减少了执行文件长度
* 方便替换使用的库函数的版本而不需要重新链接。

坏处：
* 增加了首次的运行时间

> GCC编译器，默认引用动态库，动态库是以.so结尾；而静态库是以.a为扩展名。引用一个程序库，可以使用-l*NAME*选项。


# 存储器分配

1. malloc 分配指定大小（字节数）的存储区，初始值不确定
2. calloc 为指定数量且指定长度的对象分配存储空间，初始化值为0
3. realloc 更改以前分配区的长度（增加或减少）

```

# include <stdlib.h>

void *malloc(size_t size);
void *calloc(size_t nobj, size_t size);
void *realloc(void *ptr, size_t newsize);

void free(void *ptr);

```

三个函数返回的指针一定是适当对齐的（原理基本上是已对齐单位的倍数的方式多分配内存）。

三个函数返回void *指针，因此可以将其赋值给任何指针而不需要强制转换。

free可以释放由上面三个函数分配的内存。

这些分配函数通常调用sbrk系统调用实现。

sbrk可增大和减少进程的存储空间，但malloc和free一般都不减少存储空间，以便分配并保持在malloc池中不返回内核。

# 环境变量

环境变量字符串的形式为```name=value```，unix不会查看此字符串，完全由程序解释。

ISO C定义了一个函数getenv，根据name获取其value。

```
#include <stdlib.h>

char *getenv(const char *name);
```

除了获取环境变量，还可以设置环境变量：

```

#include <stdlib.h>

int putenv(char *str);

int setenv(const char *name, const char *value, int rewrite);

int unsetenv(const char *name);
```

putenv，参数为name=value字符串，若name存在，则先删除

setenv，参数name的值设为value，如果name存在，则根据rewrite参数决定是重写还是不删除。

unsetenv，删除name的定义

> 注意：
> 1. putenv的实现，linux直接将字符串地址作为参数放入环境表中，如果参数是存放在栈中，则会发生错误。
> 2. 由于环境变量存放在程序空间的最上面，大小是有边界的，因此，如果设置的环境变量超出了存储空间大小，则需要由malloc在堆上分配空间来存储。

# setjmp和longjmp函数

c语言中，goto语句不能跨函数，如果要跨函数，则需要setjmp和longjmp函数：

```
#include <setjmp.h>

int setjmp(jmp_buf env);
void longjmp(jmp_buf env, int val);
```

调用这两个函数，遇到的问题是：

1. 调用longjmp后，自动变量和寄存器变量的状态如何？这些值能回滚到调用setjmp的状态码？

答：不确定，跟实现有关系。大多数实现并不回滚这些自动变量和寄存器变量的值，而所有标准则说他们的值是不确定的。如果有一个自动变量，而又不想使其回滚，则可以定义具有volatile属性。声明为全局或静态变量的值在执行longjmp时保持不变。

2. 自动变量的问题：若在函数内部声明指针，并返回了这个指针，则会出问题。

# 进程资源

## 进程资源使用

getrusage()系统调用返回调用进程或者其所有子进程运行所使用的各种系统资源。

```
#include <sys/resource.h>

int getrusage(int who, struct rusage *res_usage);
```

参数who指定了进程中谁的资源使用信息将会被获取。有下面几个值：

* RUSAGE_SELF 返回调用进程的资源
* RUSAGE_CHILDREN 返回调用进程的所有子进程的资源。子进程是停止的和wait的。
* RUSAGE_THREAD 返回调用线程的资源（Linux 2.6.26）

res_usage参数是一个指针，指向结构为rusage的对象。

## 进程资源限制

每个进程都消耗系统资源，OS对每个进程都有一些资源限制。在shell中，可以使用ulimit命令设置进程的资源限制。所有通过shell启动的进程都会继承ulimit设置的限制。

```getrlimit()``` 和 ```setrlimit()``` 两个函数可以获取和设置进程的资源限制。

```
#include <sys/resource.h>
int getrlimit(int resource, struct rlimit *rlim);
int setrlimit(int resource, const struct rlimit *rlim);
// Both return 0 on success, or –1 on error

```

*resource*参数代表了需要获取或设置的资源标记符。rlimit是一个结构体，用来描述资源限制的值。

```
struct rlimit {
rlim_t rlim_cur; /* Soft limit (actual process limit) */ 
rlim_t rlim_max; /* Hard limit (ceiling for rlim_cur) */
};
```

关于结构体```struct rlimit```解释如下：

既有软限制（rlim_cur），又有硬限制（rlim_max）。软限制表示进程可以消耗的最大资源设置。一个进程可以调整软限制，调整范围是[0-硬限制]。进程可以调整其硬限制。没有权限的进程，只能降硬限制调小（但不小于软限制），对于大多数进程，硬限制只是说明进程的可消耗的资源的最大值。

有权限的进程（CAP_SYS_RESOURCE）可以调整硬限制大小，既可往大得方向调整，也可以往小的方向调整。

如果rlim_cur和rlim_max的值是RLIM_INFINITY，表示没有限制。


虽然设置进程的资源限制是针对单个进程的。但进程可消耗的资源除了与这个设置有关系外，还需要依赖同一个用户id的进程所消耗的资源之和。

比如RLIMIT_NPROC资源，表示可创建的进程数限制，就是一个很好的例子。但是如果根据进程的子进程数来检查这个限制，将会失效，因为子进程也会创建新的进程。实际上，这个资源限制是针对所有的用户id都相同的进程设置的。但是，即使是具有同样的用户id，如果其他进程并没有针对这个资源进行限制，或者有限制，但是限制数跟其他不一样，则这个限制资源检查则是根据这个进程的相应设置来进行相应检查的。

下表是参数*resource*的取值表：


 resource        | Limit on          	
 ------------- | :-------------: 
 RLIMIT_AS		| 进程的虚拟内存大小（byte）
 RLIMIT_CORE	| Core 文件大小（byte）
 RLIMIT_CPU		| 进程的cpu时间(seconds)
 RLIMIT_DATA	| 进程的数据段大小（byte）
 RLIMIT_FSIZE	| 文件大小（byte）
 RLIMIT_MEMLOCK	| 锁定的内存大小（byte）
 RLIMIT_MSGQUEUE | 为实际用户id分配的POSIX的消息队列的大小(since Linux 2.6.8)
 RLIMIT_NICE	| nice的值(since Linux 2.6.12)
 RLIMIT_NOFILE	| 最大的文件描述符个数
 RLIMIT_NPROC 	| 一个实际用户id可创建的进程数
 RLIMIT_RSS		| Resident set size (bytes; not implemented)
 






