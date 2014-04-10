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




