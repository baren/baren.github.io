---
comments: true
date: 2014-02-10 19:33:00
layout: post
title: 类型和对象(PER4 chapter 3)
summary: '类型和对象'
tags:
- python
- PER(python essential reference)
---


# 对象身份和类型

内置函数```id()```以整数的方式返回对象的身份。这个整数通常指的是内存的位置，但并不保证，这依赖于实现。

```is```操作符用来比较两个对象的**身份**的。

```type()```返回对象的类型。

使用例子：

```

# Compare two objects 
def compare(a,b):
    if a is b:
        # a和b是同样的对象，也就是身份相同 
        statements
    if a == b:
        # a和b具有同样的值，身份可以不一样，只要值一样 
        statements
	if type(a) is type(b):
		# a和b具有同样的类型 
		statements

```

对象的类型：

对象的类型本身就是一个对象。这个类型对象被称为对象的类。这个类型对象是被唯一定义的，并且给定类型的所有实例的类型都是一样的。因此类型对象之间进行比较，可以使用操作符```is```进行比较。

所有的类型对象都赋予一个名字，可以用来进行类型检查。大多数这种名字都是内置的，比如list、dict等。例如：

```

if type(s) is list: 
	s.append(item)
if type(d) is dict: 
	d.update(t)

```

一个比较好的判断类型的方式是使用内置的```isinstance(object, type)```函数，因为这个函数是可以识别继承的。比如：

```

if isinstance(s,list): 
	s.append(item)
if isinstance(d,dict): 
	d.update(t)

```

# 引用计数和垃圾回收

所有的对象都是引用计数的。

只要把对象赋给一个新名字，或者存放在容器中（list、dict等），都会使引用数增加。

```
a = 37 # Creates an object with value 37 
b = a # Increases reference count on 37 
c = []
c.append(b) # Increases reference count on 37

```

当a赋值给b的时候，值为37的对象的引用数加1；当b存放在容器c中，值为37的对象的引用数加1.

整个例子中，只有一个对象保存了37的值，其它的操作仅仅是创建了一个新的指向这个对象的引用。

这就是引用计数的概念。在考虑引用计数，一定要记住，所有都是对象。

还可以通过操作来减少引用，比如：

```
del a # Decrease reference count of 37 
b = 42 # Decrease reference count of 37 
c[0] = 2.0 # Decrease reference count of 37
```

可以通过函数来查看对象的引用情况：

```
>>>
>>> a = 37
>>> import sys
>>> sys.getrefcount(a)
7
>>> sys.getrefcount(37)
9
```

当对象的引用计数到0的时候，就会被垃圾回收。


对于循环引用，其引用计数不为零，但是已经不被使用的对象，解释器会定期执行循环引用检查。发现就会回收。


# 引用和拷贝












