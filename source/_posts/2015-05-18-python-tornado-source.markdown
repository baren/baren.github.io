---
layout: post
title: "tornado源代码研究笔记"
date: 2015-03-22 15:32:01 +0800
comments: true
categories: 
- python
- tornado
---

[pipe-1]: /images/assets/pipe-1.png "linux"

主要记录研究tornado源代码的心得笔记。


<!-- more -->

# 命令行参数解析

使用tornado，通常需要解析命令行参数，来指定运行行为，比如是否开启调试模式，端口号是多少，日志文件位置等等。

tornado的命令行使用方式一般是这样：

```python

from tornado.options import define, options
define("port", default=8888, help="run on the given port", type=int)
define("mysql_host", default="127.0.0.1:3306", help="Main user DB")
define("memcache_hosts", default="127.0.0.1:11011", multiple=True,
       help="Main user memcache servers")

def connect():
    db = database.Connection(options.mysql_host)
    ...

# 后面会跟着下面语句，一般在启动函数内

tornado.options.parse_command_line()  # 解析命令行传入参数
# or
tornado.options.parse_config_file("/etc/server.conf")  # 解析文件传入参数


```

所有的定义的参数，不管是在文件内定义的，还是通过命令行传入的，都保存到全局变量options里面。

甚至在模块内，也可以有自己的参数定义。其条件是比如在调用parse_command_line函数时，import这个模块。

这样通过define函数定义的参数和通过命令行传入的参数，都会存在options变量下面。这样使用：

```python

http_server = tornado.httpserver.HTTPServer(application)
http_server.listen(options.port)
```

问题是，python自带了解析命令行参数的模块*getopt*，tornado为啥不用这个，又自己写了这样的解析模块呢？

先了解下sys.argv和getopt模块。

## sys.argv介绍

启动参数，会放在sys.argv下面，比如：

```python
# argecho.py
import sys

for arg in sys.argv:
    print arg

```

使用例子：

```
$ python argecho.py abc def     
argecho.py
abc
def
$ python argecho.py --help      
argecho.py
--help
$ python argecho.py -m kant.xml 
argecho.py
-m
kant.xml
```

sya.argv仅仅已空格区分，然后依次记录启动，其中argv[0]是启动脚本名称。

python提供了getopt来方便解析argv的参数。

## getopt模块

getopt函数支持下面几种参数定义：

```
-a
-bval
-b val
--noarg
--witharg=val
--witharg val
```

getopt有三个参数：

* 第一个参数是需要解析的参数列表，一般是从argv[1]开始：sys.argv[1:]。
* 第二个参数是短的选项定义，单个字符的。如果需要接收参数则后面跟着冒号“:”，比如：'ab:c:'，表示-a不接收参数，-b和-c后面都要跟着参数。
* 第三个参数提供长选项名字的列表。如果有参数，后面跟着“=”，比如：'noarg', 'witharg=', 'witharg2=' 













