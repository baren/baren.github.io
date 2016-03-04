---
layout: post
title: "yield（generator和coroutine）"
date: 2015-03-22 15:32:01 +0800
comments: true
categories: 
- python
- python essential reference
- yield
- generator
- coroutine
---

[pipe-1]: /images/assets/pipe-1.png "linux"

主要介绍python中yield的用法。在python中，与yield关键字相关的语法有generator和coroutine，这里会分别介绍这两种语法。



<!-- more -->

# generator

一个函数，正常情况下，会返回一个值，但是generator，返回的是值的序列，而不是一个值。在python中，用yield来实现，比如：


```python

def countdown(n):
    print("Counting down from %d" % n) 
    while n > 0:
        yield n
        print 'after yield'
        n -= 1 
    return

```

函数内使用yield关键字的函数，跟普通函数不一样，函数的调用并不会执行函数体，而是返回一个generator对象，比如：

```python
>>> a = countdown(10)  # 没有执行第一行的print语句
>>> print a
<generator object countdown at 0x10695ba00>
```

要想执行函数体，可以使用

* generator的next()函数或者
* for语句，sum()或者其它消费集合的操作。

## next()函数

generator的调用过程：

1. 当在generator上调用next()函数，会执行函数体，直到遇到yield关键字。然后
2. yield生成一个值，
3. 并且在这点，函数被挂起，直到下一个next()函数调用。然后，
4. 函数继续从yield语句后面执行。

比如：

```python
>>> a = countdown(10)
>>> print a
<generator object countdown at 0x10695ba00>
>>>
>>> a.next()
Counting down from 10
10
>>> a.next()
after yidle
9

```

当generator返回时（return），迭代停止，这时候，再调用next()函数，抛出StopIteration异常。

```python
>>> a.next()
after yidle
1
>>> a.next()
after yidle
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
```

## 迭代generator

还可以使用for语句迭代generator，比如：

for n in countdown(10): 
    print n

```python
>>> for n in countdown(10):
...     print n
...
Counting down from 10
10
after yidle
9
.
.
.
1
after yidle
>>>
```

## 停止generator

停止一个generator，通过：

* return语句返回
* 抛出一个StopIteration异常

如果generator停止了，继续调用next()函数，会抛出StopIteration异常。

## close()函数

为了防止一个generator没有停止，提供了close()函数来主动停止generator，比如：

```python
for n in countdown(10): 
    if n == 2: 
        break
    statements
```

上面程序中，for循环在生成2的时候停止了，这时候countdown并没有完全执行完成，当生成器不再调用时，调用close关闭它。

```python
>>> c = countdown(10)
>>>
>>> c.next()
Counting down from 10
10
>>> c.close()
>>> c.next()
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration

```

在generator内部，close函数调用会导致在yield语句上抛出GeneratorExit异常，因此可以在generator内部可选择的捕获这个异常（不捕获也不会抛出异常，就像上面例子中c.close()并没有出错），来做一些清理工作：

```python
def countdown(n):
    print("Counting down from %d" % n) 

    try:
        while n > 0: 
            yield n
            n=n-1 
    except GeneratorExit:
        print("Only made it to %d" % n)
```


# 协程和yield表达式

yield关键字还可以作为表达式，出现在赋值表达式右边。比如：

```python

def receiver():
    print("Ready to receive") 
    while True:
        n = (yield) 
        print("Got %s" % n)
```

以这样的方式使用yield的函数，被称作协程。

与generator一样，执行这种协程函数时，调用函数并不会执行，必须先调用next()函数或者send(None)才可以执行函数，比如：

```python
>>> c = receiver()
>>> c.next()
Ready to receive
>>> c.send(1)
Got 1
>>> c.send(2)
Got 2
>>> c.send('Hello')
Got Hello

```

协程函数执行步骤是：

1. 先调用next()，执行协程函数，函数执行到yield表达式后挂起，等待接收send函数发给它的数据
2. 调用send函数，传递给send函数的值会被协程函数内的"(yield)"表达式返回
3. 协程接收到数据，恢复执行，直到遇到下一个yield表达式

> 注意：
> 对协程函数来说，在接收数据之前，需要先执行函数体，可以使用next()和send(None)来执行。
> 如果调用send(非None值)，则会抛出TypeError错误：
> c.send(1)
> Traceback (most recent call last):
>  File "<stdin>", line 1, in <module>
> ypeError: can't send non-None value to a just-started generator
> .


与generator一样，关闭一个协程，有两种方式：

* 调用close函数
* 自己返回

一个关闭的协程函数，如果再发送数据给它，则会抛出StopIteration异常，比如：

```python

>>> c.close()
>>> c.send(1)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
StopIteration
```

调用close()函数，会在协程函数内部生成GeneratorExit异常。因此，可以捕获这个异常来做一些清理工作：

```python
def receiver():
    print("Ready to receive") 
    try:
        while True:
            n = (yield)
            print("Got %s" % n) 
    except GeneratorExit:
        print("Receiver done")
```

# throw函数

generator和协程函数，都有throw函数。原型是：

```python
throw(type, value, traceback)
```
使用例子：

```python
>>> g = countdown(10)
>>> g.next()
Counting down from 10
10
>>> c = receiver()
>>> c.next()
Ready to receive
>>> c.send(1)
Got 1
>>> g.throw(RuntimeError, 3)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 4, in countdown
RuntimeError: 3
>>> c.throw(RuntimeError, 3)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 4, in receiver
RuntimeError: 3
```

g.throw(type, value, traceback)会抛出特定的异常（参数传的异常类型），抛出异常点是函数的挂起地点，也就是yield表达式地方，或者是当还没有调用next()函数时的函数的起始点。

如果：

* 函数捕获了throw抛出的异常，并用yield生成了另一个值，那么这个值就是throw的返回值，比如：

```python
>>> def receiver():
...     print("Ready to receive")
...     try:
...             while True:
...                     n = (yield)
...                     print("Got %s" % n)
...     except Exception:
...             print("Receiver done")
...             yield 'here'
...
>>> a = receiver()
>>> a.next()
Ready to receive
>>> a.send('a')
Got a
>>> a.throw(Exception, 'exc')
Receiver done
'here'
```

throw函数与next和send一样，只是throw会抛出参数指定的异常。


# 使用yield既生成值，又接收值

函数内，可以使用yield同时接收和生成值，比如这样：


```python
>>> def line_splitter(delimiter=None):
...     print("Ready to split")
...     result = None
...     while True:
...             line = (yield result)
...             result = line.split(delimiter)
...
>>> a = line_splitter(",")
>>> a.next()
Ready to split
>>> a.send("a,b")
['a', 'b']
```


其执行步骤：
1、第一个next()调用，使函数开始执行
2、到yield语句后，返回None语句，也就是result的初始值。
3、调用send函数，接收到的值赋值给line，并继续执行语句，send()的返回值是下一个yield生成的结果。


# 使用generator和coroutine

开发一个例子，tail -f的例子，把开头为#号的过滤掉，其它原样打印。

## generator使用例子：

```python
def g_tail(the_file):
    the_file.seek(0, 2)
    while True:
        line = the_file.readline()
        if not line:
            time.sleep(0.1)
            continue
        yield line

def g_filter_line(lines):
    for line in lines:
        if not line.startswith("#"):
            yield line

def g_print_line(lines):
    for line in lines:
        print line


if __name__ == "__main__":
    f = open('/tmp/thefile')
    g_print_line(g_filter_line(g_tail(f)))
```

使用generator，每个函数，都需要接收一个iterator或者叫generator数据，来驱动整个进程执行。

## coroutine例子

```python
import functools


def crontine(method):
    @functools.wraps(method)
    def wrapper(*args, **kwargs):
        a = method(*args, **kwargs)
        a.next()
        return a
    return wrapper


def c_tail(the_file, target):
    the_file.seek(0, 2)
    while True:
        line = the_file.readline()
        if not line:
            time.sleep(0.1)
            continue
        target.send(line)


@crontine
def c_filter_line(target):
    while True:
        line = (yield)
        if not line.startswith("#"):
            target.send(line)

@crontine
def c_print_line():
    while True:
        line = (yield)
        print line
```

使用coroutine，不需要每个函数都加参数，流程驱动是通过send函数来实现。








# 参考资料
1. http://www.dabeaz.com/coroutines/Coroutines.pdf
2. https://www.python.org/dev/peps/pep-0342/
3. http://www.jeffknupp.com/blog/2013/04/07/improve-your-python-yield-and-generators-explained/
4. python essential reference chapter 5
5. http://stackoverflow.com/questions/231767/what-does-the-yield-keyword-do-in-python/231855#231855


















