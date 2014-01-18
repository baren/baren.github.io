---
comments: true
date: 2013-12-30 13:15:00
layout: post
slug: committing-to-two-svn-repositories-at-the-same-time
title: linux文件io进一步描述
summary: 'linux文件io进一步描述'
image: 'accidentally-committing-to-two-svn-repositories-at-the-same-time/svn.jpg'
tags:
- linux
---

[5-2]: /assets/Figure5-2.png "Relationship between file descriptors, open file descriptions, and i-nodes"

##原子性和条件竞争

所有系统调用都是原子执行的。原子执行避免了条件竞争。所谓条件竞争，指的是两个进程，在共享资源上操作，产生的结果依赖于两个进程的执行顺序。

###排他性创建文件

在调用open系统调用函数时，如果文件存在，则如果指定O_EXCL和O_CREAT会导致open失败。这就为进程提供了一种确保是他创建了文件的方式。
如果没有这个选项，则一般是先检查文件存在不存在，如果不存在，就创建一个文件。由于是check-if-absent方式，产生条件竞争。

###追加文件内容

如果多个进程往一个文件中追加数据（比如一个全局日志文件），通常的做法是lseek到文件末尾，然后write，比如：

{% highlight c %}
	if (lseek(fd, 0, SEEK_END) == -1)
    		errExit("lseek");
	if (write(fd, buf, len) != len) 
		fatal("Partial/failed write");

{% endhighlight %}
这也产生了条件竞争。

解决这个问题的方法，就是使用open函数的O_APPEND设置。


## fcntl()文件控制操作

函数定义：

{% highlight c %}
	#include <fcntl.h>
	int fcntl(int fd, int cmd, ...);
{% endhighlight %}

###获得打开文件的状态标记（flag）

open函数的定义是：
{% highlight c %}
int open(const char *path, int oflag, ... );
{% endhighlight %}
使用fcntl系统调用，可以获取文件或者修改文件的状态标记（标记的值是在open时由参数设置的）。

比如下面代码：
{% highlight c %}
int flags, accessMode;
flags = fcntl(fd, F_GETFL); /* Third argument is not required */ 
if (flags == -1)
    errExit("fcntl");
{% endhighlight %}

获得flags后，可以检查是否具有某种标记：
{% highlight c %}
if (flags & O_SYNC)
	printf("writes are synchronized\n");
{% endhighlight %}

但是，对于O_RDONLY (0), O_WRONLY (1), and O_RDWR (2)，使用上面这种方式判断就不行（O_RDONLY的值是0，怎么&，都是0；写功能，两个值wronly和rdwr都具有写功能，读功能，rdonly和rdwr都具有读功能）。

为了判断，可以使用O_ACCMODE(3),比如下面代码：
{% highlight c %}
accessMode = flags & O_ACCMODE;
if (accessMode == O_WRONLY || accessMode == O_RDWR)
    printf("file is writable\n");
{% endhighlight %}

还可以使用F_SETFL来修改已打开文件的标记。可修改的标记为：O_APPEND, O_NONBLOCK, O_NOATIME, O_ASYNC, and O_DIRECT，试图修改其它会被忽略。

设置标记的办法是先获取标记，再设置，比如：
{% highlight c %}
int flags;
flags = fcntl(fd, F_GETFL);
if (flags == -1)
	errExit("fcntl");
flags |= O_APPEND;
if (fcntl(fd, F_SETFL, flags) == -1)
    errExit("fcntl");
{% endhighlight %}

## 文件描述符和打开文件的关系

内核维护的三个数据结构：

*  一个进程一个文件描述符表([Array data type](http://en.wikipedia.org/wiki/Array_data_structure))
*  系统级别的打开的文件描述符的表
*  文件系统的i-node表

对于每一个进程，内核都维护了这个进程打开的文件描述符表。每一个表项包含的信息是：

*  控制操作文件描述符的标记集。其实就是一个标记close-on-exec
*  一个指向打开的文件描述符（存储在open file table中的记录，也就是系统级别的打开的文件描述符的表项）的引用。

内核维护了一个系统级别的所有的打开文件描述符（通常叫做*open file table*，其表项一般被称做open file handles）的表。
一个打开文件描述符存储了关于打开文件的所有信息，包括：

*  当前文件的偏移（read()和write()会更新这个值，或者通过lseek()指定）
*  当打开文件指定的状态标记（比如open()函数的flags参数）
*  文件访问模式（read-only，write-only或者read-write）
*  一个指向这个文件的i-node对象的引用

>  #include <sys/stat.h>

>  #include <fcntl.h>

>  int open(const char *pathname, int flags, ... /* mode_t mode */);

 > |Access mode | Description|
 > | ------------- |:-------------:|
 > |O_RDONLY    | Open the file for reading only|
 > |O_WRONLY    | Open the file for writing only|
 > |O_RDWR		| Open the file for both reading and writing|


> |File creation flags | Description|
> | ------------- |:-------------:|
> |O_CLOEXEC 	| Set the close-on-exec flag (since Linux 2.6.23)|
> |O_CREAT 		| Create file if it doesn’t already exist |
> |O_DIRECT 	| File I/O bypasses buffer cache|
> |O_DIRECTORY 	| Fail if pathname is not a directory |
> |O_EXCL 		| With O_CREAT: create file exclusively |
> |O_LARGEFILE 	| Used on 32-bit systems to open large files |
> |O_NOATIME 	| Don’t update file last access time on read() (since Linux 2.6.8) |
> |O_NOCTTY 	| Don’t let pathname become the controlling terminal |
> |O_NOFOLLOW 	| Don’t dereference symbolic links |
> |O_TRUNC		| Truncate existing file to zero length |


 > | file status flags | Description|
 > | ------------- |:-------------:|
 > |O_APPEND    | Writes are always appended to end of file|
 > |O_ASYNC    | Generate a signal when I/O is possible|
 > |O_DSYNC		| Provide synchronized I/O data integrity (since Linux 2.6.33)|
 > |O_NONBLOCK		| Open in nonblocking mode|
 > |O_SYNC		| Make file writes synchronous|


 





每一个文件的i-node包含的信息有：

*  文件类型（正常文件、socket或者FIFO）和文件权限
*  一个指向加在这个文件上的锁列表的指针
*  各种文件属性，包括文件大小，和不同操作相关的时间。

> i-node在磁盘上的描述和在内存中的描述不一样。如果在磁盘上，则记录了一个文件的持久化属性。比如类型、大小和权限等。
> i-node在内存的描述，记录了打开的文件描述符指向这个i-node节点的数目、这个i-node所在的磁盘的主要（major）和次要（minor）的ID，还记录的一些短暂的属性，比如加在文件上的锁等。

如图：
![alt text][5-2]

根据上面描述，得出下面几个隐含的意思来：

*  两个不同的文件描述符引用同一个文件，则他们共享同一个文件偏移量。
*  同样规则也适用于使用fcntl()函数获取或者修改文件状态标记（上面表格中的file status flags,比如O_APPEND, O_NONBLOCK, O_ASYNC）
*  文件描述符标记（close-on-exec）则是私有的，修改它不会影响到其他的。


