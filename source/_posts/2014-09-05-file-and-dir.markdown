---
layout: post
title: "apue(文件和目录)"
date: 2014-03-03 23:06:23 +0800
comments: true
categories: linux
---

[dir-detail]: /images/assets/dir.png "file system dir detail"
[file-detail]: /images/assets/file.png "file system file detail"

# stat系列函数

3个stat函数，位于sys/stat.h头文件中，其声明：

```
#include <sys/stat.h>

int stat(const char *restrict path, struct stat *restrict buf);
int lstat(const char *restrict path, struct stat *restrict buf);
int fstat(int fildes, struct stat *buf);

```

stat函数返回path指定的文件信息，fstat与stat一样，只是接收fd作为参数。
lstat如果参数是符号链接，返回符号链接的信息，而不是返回符号链接引用的文件的信息。

结构体struct stat成员类型是基本系统数据类型（定义了某些与实现相关的数据），其中字段的定义位于sys/types.h中。

一般struct stat的定义是：

```
dev_t     st_dev     Device ID of device containing file. 
ino_t     st_ino     File serial number. 
mode_t    st_mode    文件类型和mode（权限） 
nlink_t   st_nlink   指向这个文件的链接数
uid_t     st_uid     文件的uid. 
gid_t     st_gid     文件的组id. 
[XSI][Option Start]
dev_t     st_rdev    Device ID (if file is character or block special). 
[Option End]
off_t     st_size    For regular files, the file size in bytes. 
                     For symbolic links, the length in bytes of the 
                     pathname contained in the symbolic link. 
[SHM][Option Start]
                     For a shared memory object, the length in bytes. 
[Option End]
[TYM][Option Start]
                     For a typed memory object, the length in bytes. 
[Option End]
                     For other file types, the use of this field is 
                     unspecified. 
time_t    st_atime   Time of last access. 
time_t    st_mtime   Time of last data modification. 
time_t    st_ctime   Time of last status change. 
[XSI][Option Start]
blksize_t st_blksize A file system-specific preferred I/O block size for 
                     this object. In some file system types, this may 
                     vary from file to file. 
blkcnt_t  st_blocks  Number of blocks allocated for this object. 
```

# mode_t    st_mode字段

mode_t    st_mode成员包含：
 * 文件类型
 * 文件权限
 
## st_mode判断文件类型
unix文件类型包括：

* 普通文件、目录文件（stat.h中的宏S_ISREG()和S_ISDIR()判断）
* 块特殊文件，提供对设备（磁盘）带缓冲的访问，每次以固定长度为单位访问（S_ISBLK()）
* 字符特殊文件，提供对设备不带缓冲的访问，每次访问长度可变。系统中的所有设备，要么是块特殊文件，要么是字符特殊文件。S_ISCHR()
* FIFO，命名管道，用于进程间通信S_ISFIFO()
* 符号链接，S_ISLINK()

## 文件权限信息

每个文件有9个权限位，分成三类：

* 用户（读、写、执行）
* 组（读、写、执行）
* 其它（读、写、执行）

chmod命令可以修改文件权限，其中u、g、o分别代表用户、组和其它。

可以使用宏来判断是否有某种权限：
* S_I(R|W|X)USR
* S_I(R|W|X)GRP
* S_I(R|W|X)OTH

# 进程相关ID

介绍进程相关ID，主要用于内核对进程进行权限验证使用。

进程相关联ID有6个或者更多：

* 实际用户ID和组ID
> 标识究竟是谁，曲子登陆时口令文件的登陆项
* 有效用户ID和组ID
> 内核使用这两个来进行权限验证；一般等于实际用户ID和实际组ID
* 保存的设置用户（组）ID
> 执行一个程序时，保存了有效用户和组ID的副本，一般用在执行文件时，文件设置了set-user-id和set-group-id位时，保存有效用户ID和组ID
> 方便进程在两个用户之间切换权限
对于set-user-id和set-group-id位，stat结构体中的st_mode包含了这两位。
如果设置了文件的set-uid和set-gid，则进程的有效用户ID设置为文件的用户ID（组类似）

对文件的“设置用户ID”和“设置组ID”，可以使用S_ISUID和S_ISGID进行测试


# 文件权限验证规则

## 对文件进行权限测试
这几个文件权限，验证规则是：

* 使用名字打开一个文件时，路径中的所有目录（包括隐含的当前目录），都应该具有执行权限（目录的执行权限位通常称为搜索位）
> 目录读权限和执行权限是不一样的，读权限允许读取目录中的文件列表。当一个目录是访问路径中的组成部分时，具有执行权限可通过该目录找到文件。
* 文件读权限决定了我们可以打开文件进行读操作
* 文件写权限决定了我们可以打开文件进行写操作
* 如果open函数指定了O_TRUNC,必须具有写权限
* 删除一个文件，需对文件所在目录具有写和执行权限，对删除的文件本身，则不需要
* 6个exec函数中任何一个执行某个文件，都必须对文件有执行权限

以上权限验证发生在每当进程打开、创建和删除一个文件时，内核就进行文件访问权限测试，这种验证涉及到

* 文件所有者（st_uid, st_gid，属于文件性质）
* 进程有效ID（有效用户和组ID，属于进程性质）
* 进程附加组ID（如果支持，属于进程性质）

## 对进程进行权限测试

* 进程是超级用户（有效用户ID是0），允许访问
* 进程有效ID等于文件所有者ID，则进行上面的文件权限验证（若*所有者*适当的权限位被设置，则允许；否则拒绝）
* 进程有效组ID等于文件的组ID，则进行上面的文件权限验证（若*组*适当的权限位被设置，则允许；否则拒绝）
* 若文件其它用户适当权限位被设置，则允许；否则拒绝

以上按顺序进行验证。

# 以上，基本讲述了stat结构中的st_mode字段涉及到的知识。

# 文件所有权相关

## 新文件和目录所有权

在对进程进行权限验证时，用到了文件的用户ID和组ID，这里讲一下，创建一个新文件，赋予文件的用户ID和组ID是什么。
创建文件夹和创建文件一样。

* 新文件的用户ID设置为进程的有效用户ID

对于组ID，可以选择下列之一作为组ID：

* 新文件的组ID可以是进程的有效用户ID
* 新文件的组ID可以是它所在的目录的组ID

> 对于FreeBSD 和Mac OS X，使用的是目录组ID作为新文件的组ID
> 对于linux 2.4.22，新文件的组ID取决于它所在的目录的设置组ID是否设置，
> 如果设置组ID设置了，则为组ID，如果没设置，则为进程的有效组ID。

## 修改文件所有权

下面这几个函数修改文件的用户ID和组ID

```
#include <unistd.h>

int chown(const char *pathname, uid_t owner, gid_t group);
int fchown(int filedes, uid_t owner, gid_t group);
int lchown(const char * pathname, uid_t owner, gid_t group);

```

## access函数

操作文件时，默认是按照进程的有效用户ID和有效组ID进行权限验证。也可以对进程实际用户ID和实际组ID进行验证，使用access函数

```
#include <unistd.h>

int access(const char * pathname, int mode);
```

其中，mode参数常量，取自unistd.h

R_OK 测试读权限
W_OK 测试写权限
X_OK 测试执行权限
F_OK 测试文件是否存在

# 文件权限位相关

## 文件屏蔽字

创建文件时，文件默认的权限是多少，是由文件屏蔽字来确定的。

文件屏蔽字是与文件模式一样的，只不多，屏蔽字的权限位为1的，创建文件时，文件的对应权限位为0（关闭对应权限）。

可以使用umask函数设置进程的文件屏蔽字

```
#include <sys/stat.h>

mode_t umask(mode_t cmask);

```

cmask是

* S_I(R|W|X)USR
* S_I(R|W|X)GRP
* S_I(R|W|X)OTH

这九个常量的若干个按位“或”构成的。

对于屏蔽字，有以下几点：

* 进程创建一个文件或目录时，一定会使用屏蔽字
* 对于任何屏蔽字中为1的位，文件相应的权限位一定会被关闭（即使在open、create函数中指定了响应权限位也会被关）

同时，还需注意：

* unix系统一般不处理umask的值，在登陆是，有shell的启动文件设置一次，然后从不改变。
* 但是当创建文件时，如果想确保文件的某种权限，必须在运行时修改umask的值。
* shell的umask命令可以查看和修改shell的umask的值
* 子进程修改屏蔽字的值，不影响父进程的屏蔽字的值

shell的umask命令的值，三位八进制值，从左到右，分别代表用户、组和其它权限位。4表示读，2表示写，1表示执行。
三个权限经过“或”操作，最大值是7，表示读写执行权限都有。

最常用的是002、022和027等。

umask命令可以使用-S选项打印符号形式的屏蔽字。

## 修改文件权限

除了创建文件时，根据屏蔽字和指定的权限位为文件赋予特定权限外，还可以使用修改权限函数动态修改文件权限：

```

#include <sys/stat.h>

int chmod(const char *pathname, mode_t mode);
int fchmod(int fileds, mode_t mode);

```

chmod中的mode参数取值于`sys/stat.h`中，除了上面那九种权限，还有两个设置ID常量（S_ISUID和S_ISGID）,保存正文常量（S_ISVTX）
以及三个组合常量（S_IRWX(U|G|O)），一共15种：

* S_I(R|W|X)(USR|GRP|OTH)
——九种
* S_IRWX(U|G|O)  ——三种
* S_ISUID，S_ISGID，S_ISVTX ——三种

## 粘住位

如果一个执行文件设置了粘住位(sticky bit)，则该程序在第一次执行并结束时，其程序正文部分（机器指令部分）的一个副本仍被保存在交换区。

这样的好处是：

* 该程序下次执行时，可快速装入内存。

这么做的原因是：
* 交换区占用连续磁盘空间，可以认为是连续文件（而一般文件是分散放在磁盘各处的）
* 因此程序的正文也是连续存放的

粘住位也被称为*保存正文位（saved-text bit）*，因此也就有了S_ISVTX常量。

随着现代系统进化，这个粘住位的功能也被扩展了使用范围。

Single UNIX Specification允许设置目录的粘住位，如果对目录设置了粘住位，则只有对该目录具有写权限的用户，在满足下列条件之一时，才能
删除或更名该目录下的文件：

* 拥有此文件
* 拥有此目录
* 超级用户

/tmp目录是设置粘住位的典型候选。


# 文件长度

结构体stat中，st_size表示的是文件长度，单位是字节，这个字段只对普通文件、目录文件和符号链接有意义。

对于文件大小：

* 对于目录，文件长度通常是一个数（16或512）的倍数
* 对于符号链接，文件长度是文件名中的实际字节数，比如lib
--> usr/lib,lib的长度是7

除了st_size字段外，还有st_blksize和st_blocks两个字段。其中：

* st_blksize：是读写这个文件的最佳的块大小，用这个块大小来读文件时，所用的时间最小，标准I/O也尝试一次读写st_blksize个字节。
* st_blocks: 所分配的实际的512（系统依赖）的数量

## 文件空洞

普通文件可以包含空洞，造成文件空洞的原因是：

* 设置文件偏移量超过文件尾端
* 并写了数据

这两个会导致文件空洞。

如果文件有空洞存在，则ll和du显示的文件大小是不一样的。


# 文件截短

```
#include <unistd.h>

int truncate(const char *pathname, off_t length);
int ftruncate(int filedes, off_t length);

```

# link,unlink,remove和rename函数

任何一个文件可以有多个目录项指向其i节点，创建一个指向现有文件的链接的方法：

```
#include <unistd.h>
int link(const char *existingpath, const char *newpath);

```

这是创建一个硬链接的方法。

* link这个函数包括两个步骤：1），创建新目录项；2）增加链接计数，这两个操作原子操作
* 现在大多数的实现都不支持跨文件系统创建硬链接，也即这两个目录项应在同一个文件系统内
* 大部分实现也不支持创建目录的硬链接

为了删除一个现有的目录项，使用unlink函数：

```
int unlink(const char *pathname); // 如果pathname是一个符号链接，则取消符号链接
```

这个`unlink`函数删除此目录项，并将pathname所引用的文件的链接数减1.

当链接计数达到0，该文件内容才可被删除，若有进程打开了文件，即使其计数为0，内容也不能删除，进程退出后，才可删除。

关闭一个文件时，内核：

* 先检查打开文件的进程数
* 若进程为0，再检查链接数，若为0，则删除

## 解释i节点、目录项等

i-node节点包含了大部分文件信息，比如类型、权限什么的，单并没有包含文件名，因此

一个普通文件，具有 ：一个i-node项，一个目录项（包含文件名和i-node编号）和n个数据块。
如图：

![alt text][file-detail]

一个目录，则包含一个数据块，其数据库实际上是一个目录块，目录块中包含目录下的文件的目录项信息（目录项包括i节点和文件名）
如图：
![alt text][dir-detail]

# 符号链接

符号链接是指向一个文件的间接指针，主要是避开硬链接的一些限制：

* 硬链接要求链接和文件位于一个文件系统中
* 只有超级用户才能创建指向目录的硬链接

符号链接则无此限制。

> 注意：
> 当使用以名字引用文件的函数时，应当了解函数是否处理符号链接，也就是是否会自动跟随符号链接到达它所指向的文件。
> 若open函数参数是一个符号链接，open会跟随符号链接到达指向的文件，若符号链接指向的文件不存在，则返回错误。

创建一个符号链接：

```
# include <unistd.h>

int symlink(const char *actualpath, const char *sympath);
```

open函数打开文件时，会跟随符号链接，若想打开符号链接本身，则使用：

```
ssize_t readlink(const char *restrict pathname, char *restrict buf, size_t bufsize);
```

解除一个符号链接，可以使用unlink函数。因为unlink不跟随符号链接。

# 文件时间

与文件相关的三个时间值：

* st_atime. 文件数据的最后访问时间 比如read函数
， ls命令用-u选项查看
* st_mtime。文件数据的最后修改时间 比如write函数， ls的默认选项就是查看这个时间
* st_ctime。文件i节点状态的最后更改时间 比如chmod，chown函数， ls的-c选项查看这个时间

# utime函数

对文件访问和修改时间，可以用utime函数进行修改

```
#include <unistd.h>

int utime(const char *pathname, const struct utimebuf * times);

struct utimebuf{
time_t actime; /*访问时间*/
time_t modtime; /*修改时间*/
}
```

> 注意：
> utime并不修改st_ctime时间，当调用utime时，会自动更新

# 目录相关

## mkdir和rmdir函数

mkdir创建目录，rmdir删除目录

```
#include <unistd.h>
int mkdir(const char *pathname, mode_t mode);
int rmdir(const char *pathname);
```

对于mkdir，常见的错误是忘记设置其执行权限位，以便能访问目录中的文件名。
rmdir函数只删除空目录，空目录只包含.和..两个目录
> 注意：
> 创建目录时，目录的用户ID和组ID与文件创建时的规则一样。

## 读目录

* 对目录具有访问权限的用户都可读目录
* 但只有内核才能写目录。一个目录的写权限位和执行权限位决定了在该目录中能否创建新文件以及删除文件，它们并不代表能否写目录本身

虽然目录的实际格式依赖于具体文件系统的实现。但posix.1定义了读目录的相关接口，一般系统实现都组织read函数读目录内容

```
#include <dirent.h>

DIR *opendir(const char *pathname);

struct dirent *readdir(DIR * dp);
int clostdir(DIR *dp)
```

dirent.h中定义的dirent结构与实现相关，但至少会包含下列两个成员：

```
struct dirent{
ino_t d_ino;  /*i-node数*/
char d_name[NAME_MAX+1];
}
```
其中，NAME_MAX依赖于具体实现，并没有规定一个常量。

opendir函数返回的指向DIR结构的指针，由另外其它函数使用。
opendir执行初始化操作，使第一个readdir读目录中的第一个目录项，目录项的顺序与实现有关，通常不按照字母顺序排列。

## 工作目录

每个进程都有一个当前工作目录，此目录是搜索所有相对路径名的起点。

进程可以通过chdir和fchdir函数更改当前工作目录

```
#include <unistd.h>
int chdir(const char *pathname);
int fchdir(int fd);
```

当前工作目录是进程的一个属性，所以调用chdir只影响当前进程本身，不影响其它进程。

内核保持有当前工作目录的信息，按理应该能取其当前值，但是，内核为进程保存了指向该目录v节点的指针等目录信息，并不保存完整的路径名。

有个函数提供了获取工作目录的功能：

```
#include <unistd.h>
char *getcwd(char *buf, size_t size);
```