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

在generator内部，close函数调用会导致在yield语句上抛出GeneratorExit异常，因此可以在generator内部捕获这个异常，来做一些清理工作：

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




















# 参考资料
1. http://www.dabeaz.com/coroutines/Coroutines.pdf
2. https://www.python.org/dev/peps/pep-0342/
3. http://www.jeffknupp.com/blog/2013/04/07/improve-your-python-yield-and-generators-explained/
4. python essential reference chapter 5
5. http://stackoverflow.com/questions/231767/what-does-the-yield-keyword-do-in-python/231855#231855


















