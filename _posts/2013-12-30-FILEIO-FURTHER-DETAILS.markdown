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

