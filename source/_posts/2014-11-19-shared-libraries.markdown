---
layout: post
title: "Shared libraries简介"
date: 2014-11-19 21:09:01 +0800
comments: true
categories: 
- linux
- ld
---

[gcc-dl]: /images/assets/gcc-dl.png "linux"
[gcc-dl-u]: /images/assets/gcc-dl-u.png "linux"

共享库是把库函数放到一个单元里，在运行时供多个进程共享的一种技术。可以节省磁盘空间和内存。

主要讲静态链接和动态链接。
<!-- more -->
# 目标库（Object Libraries）

平常，编译一个程序，就是用gcc，后面罗列这所有用到的单个文件来生成一个可执行程序：

```
$ cc -g -c prog.c mod1.c mod2.c mod3.c
$ cc -g -o prog_nolib prog.o mod1.o mod2.o mod3.o
```

缺点：
* 需要在后面指定大量文件，在生产环境不适用

为了避免这个问题，可以把一些共用对象文件归到一个单一单元中，叫做目标库（object library）。

目标库（object library）有两种类型：静态（static）的和共享（shared）的。相对来说，共享的更现代，也更具有优点。

# 静态库（static library）

静态库也叫archive，是unix最开始的库类型。优点：

* 可以把预先编译好的文件放到一个单元中，避免了每次编译
* 简化了链接命令

## 创建静态库

使用ar命令来生成静态库：

```
$ ar options archive object-file...
```

## 使用静态库

两种使用静态库的方式：

1. 直接放到link的命令行里：

```
$ cc -g -c prog.c
$ cc -g -o prog prog.o libdemo.a

```

2. 静态库放到标准目录（比如/usr/lib）中，在命令行中使用“–l”选项：

```
$ cc -g -o prog prog.o -ldemo

```

如果静态库没有在标准目录中，可以指定目录：

```
$ cc -g -o prog prog.o -Lmylibdir -ldemo
```

## 缺点

* 浪费内存，每个进程都会存有一份静态库的副本。
* 如果静态库中的某个文件变动了，所有依赖这个静态库的程序，都需要重新编译。

# 共享库（Shared Libraries）

共享库解决了静态库的缺点，核心是所有依赖这个库的进程都共享同一份程序内存。当第一个依赖这个库程序运行时，会把这个库加载到内存；其它依赖这个库的程序运行时，它就已经加载到内存了。

优点：

* 由于程序小了，因此加载程序时快了
* 库不是拷贝到每个进程，因此节省了内存

不利点：

* 比静态库复杂
* 共享库必须被编译成位置无关代码（position-independent code）
* Symbol relocation必须在运行时定位，因此要稍微慢一点比静态库

# 创建和使用共享库

只考虑ELF类型的共享库

## 创建共享库

```
$ gcc -g -c -fPIC -Wall mod1.c mod2.c mod3.c
$ gcc -g -shared -o libfoo.so mod1.o mod2.o mod3.o

# 或者一条命令

$ gcc -g -fPIC -Wall mod1.c mod2.c mod3.c -shared -o libfoo.so

```

–fPIC 知道位置无关代码；

-shared指定创建共享库。

注意：
* 按照惯例，共享库前缀是lib，后缀是.so

### 位置无关代码（Position-Independent Code ）

gcc -fPIC 指示编译器生成位置无关代码（*position-independent code*）。

这对某些操作来说会改变编译器生成代码的方式。这些操作包括：访问全局、静态或者外部变量；访问字符串常量；获取函数地址等。

这些改变允许可以在运行时在任意虚拟地址定位代码。这对于共享库来说是必要的，因为在链接时刻，是不知道共享库代码在内存的具体地址的。

## 使用共享库

要使用共享库，比起静态库来，额外需要两个步骤：

* 由于最后的可执行文件，没有包含锁依赖的目标文件的拷贝，必须有个机制来在运行时标记它所依赖的共享库。这通过在链接阶段，把共享库的名字嵌入到可执行文件中。这个依赖的共享库列表叫做：dynamic dependency list。

嵌入共享库名字到可执行文件，在链接阶段是自动的：

```
$ gcc -g -Wall -o prog prog.c libfoo.so
```

* 在运行时，必须有个机制来解决嵌入的共享库名字，也就是根据名字找到共享库的文件，然后加载到内存。


根据可执行文件中嵌入的共享库名字，来寻找共享库文件，需要动态链接（dynamic linking）步骤，这是通过动态链接器（dynamic linker）来实现。动态链接器本身就是一个共享库，名字是/lib/ld-linux.so.2，这个链接器也会被嵌入到使用共享库的执行文件中。


动态链接器的任务是：根据一定预先定义的规则在文件系统中寻找共享库文件。这些规则指定了一些存放共享库的标准目录，比如，一般放在/lib和/usr/lib目录下。

### LD_LIBRARY_PATH 环境变量

除了标准目录，还可以使用环境变量LD_LIBRARY_PATH指定其它目录来通知动态链接器去这些目录寻找共享库文件。如果指定了这个共享库，则动态链接器会先从这几个目录寻找，然后再去标准目录去寻找。

## 共享库的Soname

实际上，一般不会直接让共享库的实际名字嵌入到可执行文件中，而是，在创建一个共享库的时候，会创建一个共享库的soname（在ELF中的DT_SONAME标签）。

一旦一个共享库创建了一个别名soname，那么在链接阶段，共享库的soname会被嵌入到可执行文件中。随后动态链接器会根据这个soname在运行时来寻找共享库的实际文件。

soname的目的是：

1. 允许在运行时，可以使用一个与编译时不同的共享库版本

### 创建一个具有soname的共享库：

```
$ gcc -g -c -fPIC -Wall mod1.c mod2.c mod3.c
$ gcc -g -shared -Wl,-soname,libbar.so -o libfoo.so mod1.o mod2.o mod3.o

# 共享库的soname是libbar.so
```

一旦共享库有了soname，那么使用共享库时：

```
$ gcc -g -Wall -o prog prog.c libfoo.so
```

即使使用的是共享库的真实文件名，链接器也会发现，这个共享库实际上有soname，因此，会使用soname来替代。

一旦创建了一个soname的共享库，还有一个重要步骤是创建一个soname到实际共享库的软链：

> 注意：
> 这个软链必须创建在动态链接器的查找目录中

编译步骤图示：
![alt text][gcc-dl]

使用步骤图示：
![alt text][gcc-dl-u]


# 共享库相关的工具

## ldd

ldd命令展示一个程序需要依赖的共享库：

```
$ ldd prog
libdemo.so.1 => /usr/lib/libdemo.so.1 (0x40019000) libc.so.6 => /lib/tls/libc.so.6 (0x4017b000) /lib/ld-linux.so.2 => /lib/ld-linux.so.2 (0x40000000)
```

# 共享库版本和命名规范

* minor versions：一般来说，如果如果随后的共享库的版本与上一个共享库的版本是兼容的，比如函数签名没有变，这种版本不一样但是兼容的情况，叫做共享库的 * minor versions *。

* 如果共享库的两个版本是不兼容的，则是 * major version*。

为了处理这两种差异，为共享库命名定义了一些规范。

* 共享库的每一个不兼容版本，用唯一的major version 标识符标记，这是共享库实际名字的一部分。按照约定，使用数字累加来当做major version。
* 共享库的兼容版本，使用minor version标识符来标记，minor version可以是任意字符串，但是，约定也是数字，也可以是两个数字，用句点隔开的。

根据上面约定，一个共享库的名字组成是：


libname.so.major-id.minor-id.


例子：

```
libdemo.so.1.0.1
libdemo.so.1.0.2  # Minor version, compatible with version 1.0.1 
libdemo.so.2.0.0  # New major version, incompatible with version 1.* libreadline.so.5.0
```

## soname命名规范

共享库的soname的命名规范，包括它对应的共享库实际文件的major version标识符，而不包括minor version标识符，因此soname的名字组成是：

libname.so.major-id

soname在共享库实际文件同一个目录下创建一个软链。

一般情况下，因为共享库的minor version标记这是兼容版本，因此soname一般指向最新的最小版本文件。假如libdemo.so.1这个版本下的小版本最新的是0.2，那么：

libdemo.so.1  -> libdemo.so.1.0.2

## linker name

除了这两个外，还有个linker name，这个名字只包括主要名字，不包括版本，比如：

libdemo.so -> libdemo.so.2 

libreadline.so -> libreadline.so.5


linker name 既可以指向一个实际的共享库，也可以指向soname的软链。一般是指向soname

























