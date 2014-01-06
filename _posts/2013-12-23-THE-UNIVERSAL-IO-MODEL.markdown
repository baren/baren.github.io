---
date: 2013-12-23 14:36 AM
title: FiFe I/O 通用的I/O模型
source: 
layout: post
category: linux
---


##总述
所有完成IO的系统调用，都是使用文件描述符（file descriptor）来引用一个文件。文件描述符用来引用打开的所有类型文件，包括管道、socket、设备等。每个进程都有自己的文件描述符集。

根据惯例，大多数的应用程序都希望打开三个标准的文件描述符（标准输入、输出和错误）。这三个文集描述符，一般是由在程序启动之前启动的shell程序打开的。


| 文件描述符        | POSIX 名称（在unistd.h头文件中定义）           | 描述  |
| ------------- |:-------------:| -----:|
| 0      | STDIN_FILENO | 标准输入 |
| 1     | STDOUT_FILENO      |   标准输出 |
| 2 | STDERR_FILENO     |    标准错误 |

在程序中，既可以使用数字，也可以使用posix的名称来指向这几个文件描述符。

下面是四个关键的完成IO的系统调用：
* fd = open(pathname, flags, mode) 。通过制定flags参数，open函数既可以打开文件，也可以创建文件。如果创建文件，mode参数用来指定创建文件的访问权限。如果不是用来创建文件，可以忽略mode这个文件。SUSv3规定，如果open成功，他保证使用这个进程的最小的没有使用的文件描述符。
* numread = read(fd, buffer, count)。读取fd引用的文件，最多读取count个字节，返回的是实际读取的字节数，如果到了文件末尾，返回0。
* numwritten = write(fd, buffer, count)。从buffer中，写入count个字节数据到fd所指向的文件中。返回的是实际写入的字节数，有可能小于count。
* status = close(fd) 。释放文件描述符，以及相关的内核资源

##I/O通用性
所谓通用性，是指这四个函数可以作用于所有的类型文件，来完成IO操作。这些文件包括管道、设备、socket等。

如果需要使用特定类型文件特有的功能，可以使用ioctl系统调用，它提供了所有通用功能以外的特定功能。


