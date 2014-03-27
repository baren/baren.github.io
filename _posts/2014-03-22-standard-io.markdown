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

标准I/O提供了三种类型的缓冲：

* 全缓冲。填满缓冲区后才进行实际的I/O操作。对于磁盘文件，通常是全缓冲的。在一个流上第一次执行I/O操作时，标准I/O函数通常用malloc获取需要使用的缓冲区。
* 行缓冲。当输入和输出中遇到换行符时，标准I/O库执行I/O操作。当涉及到终端时，通常使用行缓冲。

> 行缓冲有两个限制：

> 缓冲区长度是固定的，所以只要填满了缓冲区，即使没有缓冲区，也进行I/O操作

> 任何时候，只要标准I/O库从 a）不带缓冲的流，或者 b）一个行缓冲的流（但数据从内核获取）得到输入数据，那么就会造成冲洗所有行缓冲输出流。

> 从实现角度说，这么个限制是有原因的：如果从不带缓冲的流中获取数据，那么缓冲区中的没有冲洗的旧数据会对新数据造成污染。

* 不带缓冲。标准I/O库不对字符进行缓冲存储。标准错误流通常不带缓冲的。

iso c 是这样设置缓冲的：
* 当且仅当标准输入和标准输出并不涉及交互设备时，他们才是全缓冲。
* 标准错误绝不会是全缓冲。

但是标准输入输出涉及交互设备时，一般：

* 标准错误不带缓冲。
* 其它流，涉及终端设备，则是行缓冲；否则是全缓冲。

对于这些默认的缓冲情况，可以使用函数进行修改：

```
#include <stdio.h>

void setbuf(FILE * fp, char * buf);
int servbuf(FILE * fp, char * buf, int mode, size_t size);

```

这些函数的使用限制是：

* 一定要在流已被打开后调用
* 在对流执行任何一个其它操作之前调用

setbuf可以打开和关闭缓冲，打开缓冲，将buf指向一个长为BUFSIZE的缓冲区（该常量被定义在stdio.h中）。关闭缓冲，则设置buf为NULL。
setvbuf，可以精确控制缓冲类型，通过mode参数指定：

* _IOFBF	全缓冲	
* _IOLBF	行缓冲
* _IONBF	不缓冲

> 注意：
> 如果在一个函数内分配一个自动变量类的标准I/O缓冲区，则从该函数返回之前，必须关闭该流（也就是把数据冲洗一下）。
> 一般而言，应由系统选择缓冲区的长度，并自动分配缓冲区。这种情况下，关闭此流时，标准I/O库将自动释放缓冲区。

可以通过函数强制缓冲一个流：

```
#include <stdih.h>

int fflush(FILE * fp);
```

# 打开流

使用下列函数打开流：

```
#include <stdio.h>

FILE *fopen(char * pathname, char *type);
FILE *freopen(char *pachname, char *type, FILE *fp);
FILE *fdopen(int fd, char *type);

```

使用fopen函数打开一个文件，创建一个新流，并建立这个流和文件之间的连接。这个过程也会涉及到创建新文件。

其中，type参数指定对该流的读、写方式：

* ```'r'```: 只读打开文件
* ```'w'```: 只写打开，若文件存在，截断为0；否则创建新文件
* ```'a'```: 追加打开，为在文件为写而打开。文件若存在，则不改变内容。
* ```'r+'```: 读写打开文件，初始化内容不变，初始化文件位置在文件开头。
* ```'w+'```: 读写打开文件，若文件存在，截断为0.否则创建新文件
* ```'a+'```: 为在文件尾读和写而打开文件。

> 注意：

> '+'请求一个流可以读和写，当使用这种流时，在读写之间转换时，必须刷新buffer，fflush或者订货函数fseek都可以。不然内部的buffer可能不空。

freopen函数像fclos和fopen的联合。受限关闭有fp指向的流（这个过程会忽略任何错误），然后再打开pathname指向的文件，一般用于将一个指定的文件打开为一个预定义的流。

可以使用fclose关闭流：

```
#include <stdio.h>

int fclose(FILE *fp);
```

关闭一个流，关闭文件，并刷新缓冲区的输出数据，并释放由标准I/O库分配的内存。

> 注意：
> 当程序正常关闭时（调用exit函数和正常从main返回），所有未写缓冲数据的标准I/O流，都会被冲洗；所有打开的标准I/O都会关闭。

# 读和写流

一旦打开了流，就可在三中不同类型的非格式化I/O中选择，进行读写。包括：

* 每次只读一个字符的I/O，一次读写一个字符。若流是缓冲的，标准I/O会处理所有缓冲。
* 每次一行I/O，可以使用fgets和fputs，每行以换行符终止。
* direct I/O，直接I/O。fread和fwrite函数支持这种类型的I/O。每次I/O操作读或写某种数量的对象，而每个对象具有指定的长度。用于从二进制文件中每次读或写一个结构。

## 一次一个字符

下面三个函数可以用于一次读一个字符：

```

#include <stdio.h>

# 三个函数的返回值，若成功，返回下一个字符；达文件末尾或失败，则返回EOF
int getc(FILE *fp);
int fgetc(FILE *fp);

int getchar(void);

# 写入流
int putc(int c, FILE *fp);
int fputc(int c, FILE * fp);
int putchar(int c);

```

getchar 等价于getc(stdin)，而getc和fgetc的区别是前者可以实现为宏；而后者不可。这意味着：

* getc的参数不应当是具有副作用的表达式
* fgetc是函数，可得到其地址，允许fgetc地址作为参数传递给其它函数
* fgetc函数执行时间长于getc函数（宏的优势）

不管是出错，还是到达文件末尾，三个函数都返回EOF，为了区分是出错还是到达文件末尾，需要使用ferror或feof函数。

```
#include <stdio.h>

int ferror(FILE *fp);

int feof(FILE *fp);

void clearerr(FILE *fp);

```


大多数实现，为每个流在FILE对象中维持了两个标志：

* 出错标志
* 文件结束标志

从流中读取数据后，可以调用ungetc奖字符再压回流中。

```
#include <stdio.h>

# 成功，返回c，出错返回EOF
int ungetc(int c, FILE *fp);

```
关于压回，注意：

* 压回流中的字符可以再读出，但与压入顺序相反。
* 不能压回EOF。
* 到达文件末尾，还可压回字符。下次读取将返回该字符，再次读取则返回EOF。这么做的一个原因是，一次成功的ungetc调用会清除该流的文件结束标志。

## 每次一行

下面函数是每次一行函数：

```
#include <stdio.h>

# 读一行函数，成功，则返回bug，错误或到达末尾，则返回NULL
char * fgets(char * buf, int n, FILE *fp);

char * gets(char *buf);

# 写一行函数，成功，返回非负值，出错返回EOF

int fputs(char * str, FILE *fp);

int puts(char *str);

```

两个gets函数，指定了缓冲区，把读的行送入其中。gets从标准输入读，fgets从指定流读。

* fgets必须指定缓冲区的长度。此函数一直读到下一个换行符为止，但不超过n-1（放null字符），读入的数据被送入缓冲区。
* 该缓冲区一null结尾。
* 若该行的字符数超过n-1，则fgets只返回一个不完整的行，但是，缓冲区总是以null字符结尾。

> 注意：
> gets函数不推荐，由于没有指定缓冲区大小，容易造成缓冲区溢出。

写入一行函数，将一个以null符终止的字符串写入到指定流。尾端null不会被写入。

## 二进制I/O

对于直接I/O，也就是二进制I/O，提供两个函数：

```

size_t fread(void *ptr, size_t size, size_t nobj, FILE *fp);

size_t fwrite(void * ptr, size_t size, size_t nobj, FILE *fp);

```

这两个函数的常用用法：

1. 读或写一个二进制数组。比如将一个浮点数组的第2~5个元素写到一个文件上：

```
float data[10];

if (fwrite(&data[2], sizeof(float), 4, fp) != 4)
	printf("error");
```

2. 读写一个结构

```
struct {
	
	short count;
	long total;
	char name[NAMESIZE];

} item;

if (fwrite(&item, sizeof(item), 1, fp) != 1)
	printf("error");

```

> 注意：
> 对于二进制读写的问题是：只能用于同一个系统上已写的数据，原因是：
> 1. 在同一个结构中，同一个成员的偏移量可能因为编译器和系统而异。
> 2. 用来存储多字节整数和浮点值的二进制格式，在不同机器体系结构不同。

# 定位流

三种方法定位标准I/O流

* ftell和fseek，假设位置偏移量，可以使用长整型。
* ftello和fseeko，文件偏移量，使用off_t代替长整型。
* fgetpos和fsetpos，ISO C引入，使用抽象数据类型fpos_t来记录文件的位置。

```
#include <stdio.h>

long ftell(FILE *fp); # 成功，返回当前文件位置指示，出错返回-1

int fseek(FILE *fp, long offset, int whence);

void rewind(FILE *fp);

```

定位涉及到文本文件和二进制文件，由于存放格式不一样，里面许多注意点。

## 对于二进制文件

二进制文件，文件位置指示器是从文件起始位置开始度量，并以字节为计量单位。

ftell用于二进制文件时，其返回值就是这种字节位置。

为了使用fseek定位一个二进制文件，必须指定一个字节的*offset*。以及解释这种偏移量的方式。

whence的值与内核函数lseek函数相同：

* SEEK_SET表示从文件的起始位置开始
* SEEK_CUR表示从当前文件位置开始
* SEEK_END表示从文件尾端开始。

## 文本文件

对于文本文件，他们的文件位置可能不以简单的字节偏移量来度量。主要是在非unix系统中，以不同格式存放文本文件。

为了定位文本文件，*whence*一定要是SEEK_SET，而*offset*的值只有两种：0（绕回到文件的起始位置）。或是对该文件调用ftell返回的值。

使用fewind可以将流设置到文件的起始位置。


# 格式化I/O

## scanf函数

有以下几个scanf函数：

```
#include <stdio.h>

int fscanf(FILE *stream, char const *format, ...);

int scanf(char const *format, ...);

int sscanf(char const *string, char const *format, ...);

```

这些函数都是从输入源中读取字符并根据format字符串给出的格式代码对他们进行转换。

fscanf输入源就是作为参数给出的流，scanf从标准输入读取，sscanf从第一个参数所给的字符串中读取字符。

读取停止条件：

1. 当格式化字符串到达末尾
2. 读取的输入不再匹配格式字符串所指定的类型时

返回结果是被转换的输入值的数据。

如果在任何输入值被转换之前文件就已经到达尾部，函数就返回常量值EOF。

> 注意：
> 对于scanf函数的参数，前面需要使用```&```。

scanf函数族中的format字符串参数可能包含下面内容：

* 空白字符————她们与输入中得零个或多个空白字符匹配，在处理过程中被忽略。
* 格式代码————她们指定函数如何解释接下来的输入字符。
* 其它字符————当任何其它字符出现在format字符串时，下一个输入字符必须与它匹配。如果匹配，该输入字符随后就被丢弃。如果不匹配，函数就不再读取直接返回。

格式代码的结构是：

```
%[*] [fldwidth] [lenmodifier]convtype
```
针对上面的格式，解释如下：

* 可选```*```字符：将使转换后的值被丢弃而不是进行存储
* fldwith宽度：非负整数，它限制将被读取用于转换的输入字符的个数。如果没有给出宽度，函数就连续读入字符直到遇到输入中的下一个空白符。
* lenmodifier限定符：用于修改有些格式代码的含义

> 对于限定符，说明：
> 限定符的目的是为了指定参数的长度。如果整型参数比缺省的整型值更短或更长时，在格式化代码中省略限定符就是一个常见的错误。
> 对于浮点型也是如此。
> 如果省略了限定符，可能导致一个较长的变量只有部分被初始化。或者一个较短变量的临近变量也被修改，这都取决于这些类型的相对长度。

限定符作用到整型和浮点型的效果如下表：

 格式代码        | h           | l  | L	
 ------------- | :-------------: | :---------------: | :------------------: 
 d,i,n      | short | long |	空	|
 o,u,x      | 无符号 short      |       无符号 long |	空	
 e,f,g |     空  |    double |       long double	










