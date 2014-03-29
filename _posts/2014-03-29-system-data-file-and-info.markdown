---
comments: true
date: 2014-03-03 23:06:00
layout: post
title: apue(系统数据文件和信息)
summary: 'apue chapter6'
tags:
- linux
- 笔记
---

[time-func]: /assets/Figure6-1.png "time-function"

# 时间和日期

unix内核提供的*基本时间服务*是计算自国际标准时间公元1970年1月1日00:00:00以来经过的秒数。

这种秒数是以数据类型time_t表示的。称之为**日历时间**，日历时间包括日期和时间。

time函数返回当前的时间和日期：

```

#include <time.h>

time_t time(time_t *calptr);

```

时间值总是作为函数值返回。如果参数不为空，则时间也存放在由calptr执行的内存中。

gettimeofday函数提供了更高分辨率（最高为微妙级）。

```
#include <sys/time.h>  ## 这里就是sys/time.h头文件了

int gettimeofday(struct timeval *restrict tp, void *restrict tzp);


```
该函数作为XSI扩展定于在Single UNIX Specification中，tzp唯一合法的值是NULL，其它值则将产生不确定的结果。

gettimeofday函数把当前时间放在tp指向的timeval结构中，该结构存储秒和微妙。

```
struct timeval {
	time_t tv_sec;  /*秒*/
	long tv_usec;  /*微妙，注意是微妙，不是毫秒*/

}
```

取到time_t后，可以调用其它函数来转成成可读的日期和时间。

![alt text][time-func]

localtime和gmtime将日历时间转换成以年、月、日、时、分、秒、周日表示的时间，并将这些存放在tm结构中：


```
#include <time.h>

# 将日历时间转换成国际标准时间的年、月、日、时、分、秒、周日。
struct tm *gmtime(const time_t *calptr);
# 将日历时间转换成本地时间（考虑本地时区和夏令时）
struct tm *localtime(const time_t *calptr);

struct tm{
	int time_sec; /*秒 [0-60]*/
	int time_min; /*分 [0-59]*/
	int tm_hour; /*时 [0-23]*/
	int time_mday; /*日 [0-31]*/
	int tm_mon; /*月 [0-11]*/
	int tm_year; /*年 [1900年开始]*/
	int tm_wday; /*从周日开始的天 [0-6]*/
	int tm_yday; /*从一月开始的天数 [0-365]*/
	int tm_isdst; /*daylight(夏令时)保存时间标记，<0, 0, >0*/
}
```

秒可以超过59的原因是可以闰秒。如果夏令时生效，夏令时标志值为正；如果非夏令时制时间，标志位0；不可用，则为负。

mktime函数把年月日等作为参数，转换成time_t的值：

```
#include <time.h>

time_t mktime(struct tm *tmptr);

# 下面两个函数把时间转换成date命令的系统默认输出形式。只是参数不一样
char *asctime(const struct tm *tmptr);
char *ctime(const time_t *calptr);

```

strftime函数是格式化时间：

```
#include <time.h>

size_t strftime(char *restrict buf, size_t masxize, const char *restrict format, const struct tm *restrict tmptr);

```

最后一个参数是要格式化的时间值，tm结构指针传递。格式化结果存放在一个长度为maxsize的buf数组中，如果buf长度可以存放格式化结果和一个null终止符，则该函数返回在buf中存放的字符数，否则该函数返回0.

format是格式化字符串。


转换符略。












