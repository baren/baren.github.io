---
layout: post
title: "函数和函数式编程"
date: 2014-09-13 17:07:23 +0800
comments: true
categories: 
- python
- python essential reference
---

# 作用域规则

每当一个函数执行，都会创建一个局部命名空间（local namespace）。这个命名空间代表了一个局部环境，这个环境包含了凡是参数的名字、在函数内定义的局部变量等。当解析一个名字时，解析器会：

* 首先检查局部环境变量
* 如果在局部命名空间中找不到，然后再往上寻找全局命名空间（global namespace），全局命名空间是函数定义所在的模块。
* 如果在全局命名空间不存在，解释器会查找内置的命名空间，如果还找不到，则NameError异常抛出

python的命名空间的一个特点就是：在函数内处理全局变量。

比如：

<!-- more -->

```python

a = 42 
def foo():
	a = 13 
foo()
# a 还是 42

```

这里，虽然在函数foo内给a赋值为13，但是：*当一个变量在函数内赋值时，结果总是把这个变量绑定到局部命名空间中*，因此在foo内，a与全局变量的a是不一样的。

如果要在函数内修改全局变量，必须使用*global*关键字来修饰变量。表示的意思是简单的生命这个名字属于全局命名空间。这只在这个变量需要被修改的情况下。
例子：

```python

a = 42
b = 37 

def foo():
	global a  # 'a' 此时在全局命名空间中
	a = 13 
	b=0

foo()
# a 是 13. b 仍然是 37.

```

## 嵌套函数中的变量作用域

python支持嵌套函数定义，也就是在函数内定义函数：

```python

def countdown(start): 
	n = start
	def display(): # Nested function definition 
		print('T-minus %d' % n)
	while n > 0: 
		display()
		n -= 1

```

当作用域涉及到嵌套函数时，变量的搜寻是按照*词法范围（lexical scoping）*来查找的。意思是：

* 变量名字受限检查局部范围
* 然后是函数定义处外围范围，依次往外查找，从最内层到最外层范围
* 然后就是全局命名空间
* 最后是内置的命名空间

在给变量赋值时，牵扯到嵌套函数，python2有个限制是：

* 只允许在最内层范围的和全局范围的（使用global）的变量可以被赋值。

这意味着：

* 内部函数不能够给一个局部变量赋值，这个局部变量是定义在内部函数的外围函数内。

比如：

```
def countdown(start):
	n = start
	
	def display():
		print('T-minus %d' % n)
	
	def decrement():
		n -= 1 # 在Python 2是错误的，这里不能修改这个值。
	
	while n > 0: 
		display()
		decrement()
```

在python3中，可以使用*nonlocal*关键字来修改：

```python

def countdown(start):
	n = start
	
	def display(): 
		print('T-minus %d' % n) 

	def decrement():
		nonlocal n
		n -= 1  # 绑定到外围的 n (Python 3)
	
	while n > 0: 
		display()
		decrement()

```

如果一个局部变量在赋值前被使用，则会抛出UnboundLocalError的错误，比如：

```python

i=0
def foo():
	i = i + 1 # 导致UnboundLocalError 异常 
	print(i)

```

在这个例子中，变量*i*被定义为局部变量（在函数内被赋值，并且没有global关键字声明），这样，在执行 i = i +1 语句时，会试图读取变量“i”的值，会导致错误。

# 函数作为对象和闭包（closure）

## 函数作为对象

python中，函数是一级对象（first-class），意思是

* 函数可以作为参数传递给其它函数
* 放在数据结构中
* 以及作为结果从函数中返回。


注意：
> 当函数作为数据被处理时，需要注意的是：函数隐含的携带了在函数定义处的外围环境信息。这会影响到自由的变量在函数内怎么被绑定。

比如下面例子：

```python foo_test.py

x = 33
def print_x():
    print x

```

然后在foo.py文件中导入foo_test：

```python foo.py

x = 42

def foo(func):
    func()


if __name__ == "__main__":
    foo(foo_test.print_x)

// 输出结果是33，而不是此处定义的42
```

在这个例子中，需要注意的是print_x使用的x是他所定义处的x的值，而不是在执行处的值，虽然在执行的地方，也定义了同样的变量。


## 闭包

闭包的定义：
组成函数的语句，与执行的环境绑定在一起，产生的对象，被称为闭包。可以简单的认为闭包是一个函数，它在词法上下文中引用了自由变量，所谓自由变量就是除局部变量以外的变量。

闭包只是在形式和表现上像函数，但实际上不是函数。函数是一些可执行的代码，这些代码在函数被定义后就确定了，不会在执行时发生变化，所以一个函数只有一个实例。闭包在运行时可以有多个实例，不同的引用环境和相同的函数组合可以产生不同的实例

一般有闭包特征的语言有下面这样的特性：

* 函数是第一级对象（first-class）
* 函数可以嵌套定义。

当嵌套函数被使用时，闭包会捕获嵌套函数执行所需要的整个环境。比如：

```python

import foo 
def bar(): 
	x = 13

def helloworld():
	return "Hello World. x is %d" % x
	foo.callf(helloworld) 	# returns 'Hello World, x is 13'

```

闭包和嵌套函数在你想延迟计算时，比较有用：

```python

from urllib import urlopen
# from urllib.request import urlopen (Python 3) 

def page(url):
	
	def get():
		return urlopen(url).read()
	return get

```

执行如下：

```python

>>> python_url = page("http://www.python.org") 
>>> jython_url = page("http://www.jython.org") 
>>> python_url
<function get at 0x95d5f0>
>>> jython
<function get at 0x9735f0>
>>> pydata = python()  # 读取url内容
>>> jydata = jython()  # 读取url内容
```

对于上面这个闭包例子，解释如下：

* 两个变量python_url和jython_url实际上是两个不同的get函数的版本（语句与外围环境的绑定）
* 即使创建这两个变量的函数page不再运行，get函数暗中的携带了get函数使用的外围get函数定义处的变量。
* 当get调用时，绑定了代码和外围的变量。


闭包一个非常有用的方式是：*可以用来保存跨越一系列函数调用的状态（函数式编程所用到的状态的保存）*，比如：

```python

def countdown(n): 
	def next():
		nonlocal n  # python3
		r=n
		n -= 1 
		return r
	return next

# Example use
next = countdown(10) 
while True:
	v = next()  # 获取下一个值
	if not v: 
		break

```


在这段代码中，闭包被用来保存内部的计数值 n。这样，每次调用next()函数时，都能够更新和返回上一个计数变量的值（感觉就是更新了一个全局变量，但是这个n实际上是一个局部变量）。

假设，语言不支持闭包，那么要实现上面同样的功能，可以这样(基本原则是在函数外定义一个变量，变量的生命周期不随函数调用而结束)：

```python

class Countdown(object): 
	
	def __init__(self,n):
		self.n = n 

	def next(self):
		r = self.n 
		self.n -= 1 
		return r

# Example use
c = Countdown(10) 

while True:
	v = c.next() # 获取下一个值
	if not v: 
		break

```

总结：
> 闭包可以保存变量的状态，这个变量不随着闭包调用结束而丢失。重要的特性。
> 闭包可以捕获嵌套函数的环境的特性，在包装一个存在的函数来增加额外的功能的时候，非常有用（装饰器）。

# 装饰器



























