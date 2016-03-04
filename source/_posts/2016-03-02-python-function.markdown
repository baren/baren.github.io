---
layout: post
title: "python 函数知识"
date: 2016-03-02 19:32:01 +0800
comments: true
categories: 
- python
---

# 函数命名空间

每次函数执行，都会创建一个 local namespace。用来存放在函数内定义的变量。

在函数内引用一个变量，python查找这个变量的顺序是：
	
* 局部空间
* 全局空间（函数定义所在的模块定义的变量） 
* built-in namespace

如果找不到，则抛出NameError。

如果想要在函数内修改全局变量值，则需要用global关键字声明，比如：

```python

a = 42
b = 37 
def foo():
	global a   # 'a' is in global namespace
	a = 13 
	b=0


foo()
# a is now 13. b is still 37.
```

## 内嵌函数的命名空间

函数内也可以定义函数，这叫nested function。内嵌函数的包围函数被称作外围函数（outer function）。在内嵌函数内绑定的变量是定义在词法空间（lexical scoping），在内嵌函数内，引用变量的顺序是：

* lexical scoping
* 局部空间
* 全局空间（函数定义所在的模块定义的变量） 
* built-in namespace

在内嵌函数内，可以访问包围其函数内定义的变量（对内嵌函数来说，外围函数定义的变量被称作自由变量 free variables），但是不能再内嵌函数中重新绑定其外围函数定义的变量。比如：

```python

def test():
    b = 2
    def inner():
        b -= 1
    inner()

if __name__ == "__main__":
    test()

结果是：
UnboundLocalError: local variable 'b' referenced before assignment
```

只能在内嵌函数的最外层函数和全局空间内更改变量（python3用nonlocal关键字解决这个问题）。

## 闭包

函数是first-class对象，因此，函数可以：

* 赋值给变量
* 作为参数传递给函数
* 作为函数结果返回
* 存放在数据结构中

当把函数作为数据处理时，函数会隐含的带着一些额外的相关信息，这些相关信息就是函数定义处的一些数据。

比如：

```python
# foo.py
x = 42
def callf(func):
	return func()
```

观察行为

```python

>>> import foo
>>> x = 37
>>> def helloworld():
...		return "Hello World. x is %d" % x
...
>>> foo.callf(helloworld)  # Pass a function as an argument 'Hello World. x is 37'
>>>

```

这里结果是helloworld定义位置的x的值，而不是执行位置的x的值。

闭包定义：

闭包是在其词法上下文（内嵌函数内部）中引用了自由变量（外围函数定义的变量）的函数。主要用途是当有需要延迟计算的情况，用闭包更简洁。

比如：

```python
from urllib import urlopen
# from urllib.request import urlopen (Python 3) 

def page(url):
	def get():
		return urlopen(url).read()  # 引用了自由变量url，这个值取自定义函数的时候
	return get

```

使用闭包例子：

```python
>>> python = page("http://www.python.org") 
>>> python
<function get at 0x95d5f0>
>>> jython = page("http://www.jython.org")
>>> jython
<function get at 0x9735f0>
>>> pydata = python()	# Fetches http://www.python.org
>>> jydata = jython() 	# Fetches http://www.jython.org
>>>
 
```

看上面的例子，当调用page函数时，返回的是内嵌函数，并没有立即执行，但是打开的url是固定的了。


























