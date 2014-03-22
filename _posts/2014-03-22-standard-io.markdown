---
comments: true
date: 2014-03-03 23:06:00
layout: post
title: apue(标准I/O库)
summary: 'apue chapter5'
tags:
- linux
- 笔记
---
[dir-detail]: /assets/dir.png "file system dir detail"
[file-detail]: /assets/file.png "file system file detail"

# 流和FILE对象

## 流介绍

对于标准I/O库，他们的操作是围绕*流*进行的。

与JAVA使用Stream类来代表流不一样，标准I/O采用结构体FILE来代表流，由于大多数标准库的函数使用FILE *来作为参数，因此也用*文件指针*来代称流。

struct FILE结构体在stdio.h中做了声明：

```
typedef struct _sFILE{...} FILE;

```
数据类型FILE
	> 代表流对象，FILE对象含有所有的连接文件的内部状态，包括文件位置标记和缓存信息等。
	> 每一个流海包含error和文件末尾状态标记，可以通过ferror和feof函数进行测试。

## 流的定向

标准I/O可以用于单字节的字符集（ascii字符集）和多字节（宽）字符集，这就要求*流的定向*。
对于宽字节的处理，有一套专门的多字节处理函数（wchar.h中声明）。
对于同样的流，为了能处理宽的和正常的字符的操作，有一个限制：

* 流要么被标记为宽定向的，要么是字节定向的
* 一旦确定定向后，不能二者混用

有三种方式来定向流：

* 创建流后，任一个正常字符函数被使用，流被标记为字节定向的。
* 创建流后，任一个宽字符函数被使用，流被标记为宽定向的。
* 使用```fwide函数```来设置流的定向。
* freopen函数可以清除流的定向。

注意：

* 流刚被创建时，并没有定向
* 在未定向流上使用字节字符函数则置为字节定向的
* 在未定向流上使用宽字节函数则置为宽字节定向的

fwide函数说明：

```
#include <stdio.h>
#include <wchar.h>

# 返回值：若流是宽定向的，则返回正值；若流是字节定向的，则返回负值；若流是未定向的，则返回0.
int fwide(FILE *fp, int mode);
```

根据mode参数的值，fwide执行不同的工作：

* 若mode参数为负值，fwide试图使指定的流是字节定向的
* 若mode参数为正值，fwide试图使指定的流是宽定向的
* 若mode参数为0，fwide将不试图设置流的定向，但返回标识该流定向的值

> 注意:
> fwide不会改变已经设置定向的流
> fwide不会出错，想检查错误，唯一可用的是调用之前先清除errno，调用fwide后检查errno的值。（errno是全局变量，每次调用失败，系统会用错误代码赋值给errno）

# 标准输入/输出/错误

当main函数被调用，就已经有3哥预先定义的打开的流可以使用，这三个流在stdio.h中声明：

* FILE *stdin。标准输入流
* FILE *stdout。标准输出流
* FILE *stderr。标准错误流

在GNU C中，stdin、stdout和stderr是正常常量，可以直接用：

```
fclost(stdout);
stdout = fopen("filepath", "w");

```

# 缓冲





