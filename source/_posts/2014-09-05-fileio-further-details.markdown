---
layout: post
title: "linux文件io进一步描述"
date: 2013-12-30 13:15:19 +0800
comments: true
categories: linux
---


[5-2]: /images/assets/Figure5-2.png "Relationship between file descriptors, open file descriptions, and i-nodes"
[5-3]: /images/assets/Figure5-3.png "Relationship between readv iov and iovcnt parameter"

##原子性和条件竞争

所有系统调用都是原子执行的。原子执行避免了条件竞争。所谓条件竞争，指的是两个进程，在共享资源上操作，产生的结果依赖于两个进程的执行顺序。

###排他性创建文件

在调用open系统调用函数时，如果文件存在，则如果指定O_EXCL和O_CREAT会导致open失败。这就为进程提供了一种确保是他创建了文件的方式。
如果没有这个选项，则一般是先检查文件存在不存在，如果不存在，就创建一个文件。由于是check-if-absent方式，产生条件竞争。

<!-- more -->

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


## dup文件描述符

使用shell的I/O重定向语法 *2>&1* 时，会通知shell，我们想要标准错误输出，输出到与标准输出到同一个地方。shell是从左到右计算I/O导向的。因此，下面的shell：

```
$ ./myscript > results.log 2>&1

```

会将标准输出输出到与标准输出同一个位置，也即results.log。

为了实现这个效果，使用dup和dup2两个函数。

> 如果不使用这两个函数，简单的打开results.log两次，一次使用文件描述符1，一次使用文件描述符2.这是不够的。因为从上面图可以看出，打开两次文件，这两个文件描述符并不共享同一个文件的offset，因此两个会相互覆盖。另一个原因是打开文件不一定是磁盘文件，比如$ ./myscript  2>&1 | less


下面是dup的声明：

```
#include <unistd.h>

int dup(int oldfd);

```

dup函数接收一个已经打开的文件描述符oldfd，返回一个指向同一个文件的新的文件描述符。如果失败，则返回-1.返回的新的文件描述符保证使用最小的未使用的文件描述符。

为了实现上面shell的命令，可以这样：

```
/*前提是文件描述符0已经被使用*/
close(2);
int newfd = dup(1)

```

为了方便实现这个功能，还有个dup2函数：


```
#include <unistd.h>

int dup2(int oldfd, int newfd);

```

dup2函数使用给定的newfd来复制已经打开的旧的文件描述符oldfd。返回的是新的文件描述符。

还可以使用fcntl函数来完成dup命令，保证使用的新文件描述符大于等于指定的文件描述符：

```
newfd = fcntl(oldfd, F_DUPFD, startfd);
```

这在保证在特定范围内dup文件描述符是有用的。


##  pread()和pwrite()在指定offset进行I/O操作

使用pread和pwrite，可以在指定的offset完成读和写操作，而不会影响文件的原始offset的值。

```
#include <unistd.h>
/*返回读得字节数，如果到EOF，为0；如果错误，返回-1*/
ssize_t pread(int fd, void *buf, size_t count, off_t offset);

/*返回写入的字节数，-1表示错误*/
ssize_t pwrite(int fd, const void *buf, size_t count, off_t offset);

```

> 使用pread和pwrite函数，传入的文件描述符必须是可seek的。

> 注意：
> 使用pread和pwrite在多线程环境下，可以避免条件竞争。如果使用lseek、write这种方式写文件，会产生条件竞争。使用pwrite，则多个线程会避免条件竞争。



## Scatter-Gather I/O: readv() 和 writev()

readv和writev函数定义：

```
#include <sys/uio.h>

ssize_t readv(int fd, const struct iovec *iov, int iovcnt);

ssize_t writev(int fd, const struct iovec *iov, int iovcnt);

```

与read只读取数据到一个buffer不同，readv可以一次性把读取的数据分散（scatter）到多个buffer中。iov是一个数组，数组的每个元素是一个结构，结构类型是struct iovec。buffer的个数有iovcnt指定。

struct iovec的定义是：

```
struct iovec {
	void *iov_base; /* 缓存的起始地址 */
	size_t iov_len; /* Number of bytes to transfer to/from buffer */
};

```

下图描述了iov参数和iovcnt参数的关系：
![alt text][5-3]

### 分散输入（scatter input）

readv系统调用完成了*scatter input*，读取由fd指定的文件的持续的字节序列，然后顺序的把这些数据写入到由iov参数指定的buffer中。所有的这些buffer，从iov[0]开始，会被完全的写满之后，才会继续写入下一个buffer。


注意：
一个readv的重要的属性是，这些都是完全自动的。从调用者角度，内核会把一连续的字节序列写入到buffer中。意味着，如果从一个文件中读取数据时，能够确保读入的数据是连续的，即使在这其间，有其它线程试图修改同一个文件的offset，也不会影响。

readv返回读取的数据的字节数。调用者需要自己检查一下。

### 聚集输出（gather output）

writev系统调用完成了*gather output*。参数意义与readv类似。

与write一样，writev也可能只写入部分，因此需要检查是否请求的数据全部被写入了。

### writev和readv的原因

主要原因还是

	1. 易用性
	2. 性能

使用场景，比如：
	* 如果需要调用一系列的write函数来输出buffer数据时

## 截断一个文件truncate()和ftruncate()
截断一个文件，两个函数定义如下：

```
#include <unistd.h>
int truncate(const char *pathname, off_t length); 

int ftruncate(int fd, off_t length);
```

这两个函数不会对文件的offset有影响。

## 非阻塞I/O

当打开一个文件，指定O_NONBLOCK标记时，主要有两种目的：

* 如果文件不能立即被打开，open()函数会返回错误，而不是一直阻塞着。
* 如果open()打开成功，随后的I/O操作仍然是非阻塞的。

非阻塞模式可以用于设备、FIFO（命名管道，用于Linux进程通信）、socket。由于管道和socket的文件描述符
不能够通过open()获得，因此我们必须使用 fcntl() F_SETFL操作来使这个标记可用。


注意：
> O_NONBLOCK对于普通文件是忽略的。因此linux的内核的buffer确保了普通文件的I/O是非阻塞的。

## I/O大文件

linux使用off_t数据类型来存储文件的offset，off_t使用有符号的长整型来描述（之所以有符号，是方便用-1表示失败）。
因此在32位机器上，一个文件的最大限制是2^31-1 byte。

为了在32位机器上实现大文件操作，厂商提供了Large File Summit (LFS)概念。linux自从2.4开始支持LFS。为了支持大文件，文件系统也需要支持。大部分linux文件系统都支持（微软的VFAT and NFSv2都不支持）

为了在linux操作大文件，有两种方法：

* 采用支持大文件的替换的API，也就是ransitional LFS API。
* 在编译我们代码时，定义宏_FILE_OFFSET_BITS的值为64.这种方式是推荐的方式。

### 使用LSF API

采用过渡的LFS API时，在编译时，需要测试_LARGEFILE64_SOURCE的宏（命令行或者源文件）。这组API可以处理64位的文件大小和offset。这组api的函数名字后面都有64.比如：

```
fopen64(), open64(), lseek64(), truncate64(), stat64(), mmap64(), and setrlimit64()
```

### 使用_FILE_OFFSET_BITS

推荐的方式是定义_FILE_OFFSET_BITS宏的值为64.
这会自动的替换32位函数和数据类型到64位的函数和数据类型。这意味这，我们可以重新编译以前写好的文件来支持大文件。

只遗留了一个问题，就是打印off_t 的值的时候，需要使用lld%来正确的显示off_t的值。

```
#define _FILE_OFFSET_BITS 64

off_t offset; /* Will be 64 bits, the size of 'long long' */ 

/* Other code assigning a value to 'offset' */
printf("offset=%lld\n", (long long) offset);
```

## /dev/fd 目录

对于每一个进程，内核提供了一个虚拟目录/dev/fd ，这个目录包含的文件形式是*/dev/fd/n*。n对应的进程中的打开的文件描述符。因此使用/dev/fd/n或者fd都可以指向一个文件。

但是在程序内很少使用这种方式，一般只在shell中使用。