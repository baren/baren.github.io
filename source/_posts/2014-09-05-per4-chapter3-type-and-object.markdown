---
layout: post
title: "类型和对象(PER4 chapter 3)"
date: 2014-02-10 19:33:16 +0800
comments: true
categories: python
---



# 对象身份和类型

内置函数 *id()* 以整数的方式返回对象的身份。这个整数通常指的是内存的位置，但并不保证，这依赖于实现。

*is* 操作符用来比较两个对象的*身份*的。

*type()* 返回对象的类型。

使用例子：

```python

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

对象的类型本身就是一个对象。这个类型对象被称为对象的类。这个类型对象是被唯一定义的，并且给定类型的所有实例的类型都是一样的。因此类型对象之间进行比较，可以使用操作符 *is* 进行比较。

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

拷贝分浅拷贝和深拷贝。

copy模块提供了浅拷贝和深拷贝操作

```
copy.copy(x)
	Return a shallow copy of x.

copy.deepcopy(x)
	Return a deep copy of x.
```


浅拷贝和深拷贝的区别只与组合对象相关（包含其它对象的对象，比如list等）

* 浅拷贝构建了一个新的组合对象，然后把从原对象中发现的引用插入到新组合对象中
* 深拷贝构建了一个新的组合对象，然后递归的把原对象的拷贝插入到新对象中

浅拷贝创建一个引用，包含的元素是原对象包含的元素的引用。比如：

```
>>> a = [1,2,[3,4]]
>>> b = list(a)
>>> b is a
False
>>> b.append(100)
>>> b
[1, 2, [3, 4], 100]
>>> a
[1, 2, [3, 4]]
>>> b[2][0] = -100
>>> b
[1, 2, [-100, 4], 100]
>>> a
[1, 2, [-100, 4]]

```

对dict的浅拷贝，可以使用 dict.copy()，对于list的浅拷贝，可以使用```copied_list = original_list[:]```

深拷贝可以使用copy模块下的deepcopy函数实现：

```
>>> import copy
>>> a = [1, 2, [3, 4]]
>>> b = copy.deepcopy(a)
>>> b[2][0] = -100
>>> b
[1, 2, [-100, 4]]
>>> a # Notice that a is unchanged [1, 2, [3, 4]]
>>>

```

如果一个类想定义自己的浅拷贝和深拷贝实现，可以实现特殊方法``` __copy__() and __deepcopy__()```。前者是在浅拷贝的时候调用（copy.copy(x)）。后者是在深拷贝的时候调用(copy.deepcopy(x))。

# 第一类对象

第一类对象是指可以在执行期创造并作为参数传递给其他函数或存入一个变量的实体。一般第一类对象具有的特征是：

* 可以被存入变量或其他结构
* 可以被作为参数传递给其他函数
* 可以被作为函数的返回值
* 以在执行期创造，而无需完全在设计期全部写出

在python中，所有的对象都是第一类对象。

所有对象都是第一类对象的好处是可以写出很紧凑简洁的代码。


# 表示数据的内置类型

略

# 表示程序结构的内置类型

## 可调用类型

可调用类型表示对象支持函数调用操作。包括用户定义函数，内置函数、实例方法和**类**。

### 用户定义函数

在模块级别创建的，通过def或者lambda操作符创建的用户定义的函数，是可调用的对象。

比如：

```

def foo(x, y):
	return x + y

foo = lambda x,y: x + y


```

用户定义函数的属性包括：

描述        | 描述           	
------------- | :-------------: 
f._ _dict_ _ | 包含函数属性的dict
f._ _defaults_ _ | 包含默认参数的元组


```
>>> def foo(x=1, y=1):
...     return x + y
...
>>> foo.__defaults__
(1, 1)
```

用户定义的函数的类型是```types.FunctionType```

比如：

```

>>> def foo(x=1, y=1):
...     return x + y
...

>>> import types
>>> type(foo) is types.FunctionType
True
>>> type(foo) is types.MethodType
False
```

### 方法

方法是定义在类内部的函数。有三种方法：

* 实例方法——对对象实例上操作的函数
* 类方法——类本身作为一个对象，类方法是在类本身操作的函数
* 静态方法——仅仅是个函数，不接收类本身或类的实例作为参数

比如：

```
class Foo(object):
	# 实例方法，第一个参数self
	def instance_method(self, arg):
		statements 

	# 类方法， 第一个参数cls
	@classmethod
	def class_method(cls, arg): 
		statements

	# 静态方法，没有self或cls的参数
	@staticmethod
	def static_method(arg):
		statements
```

实例方法和类方法都由types.MethodType来表示。

详细了解对象查询属性操作（.）是如何工作的，对理解这种特殊类型是有好处的。

在一个对象上查询什么（.）与函数调用总是分开的，当调用方法时，两个操作都会发生，只是步骤差别。



1. 在实例上查询

例子：

```
f = Foo() # 创建一个实例

meth = f.instance_method # 查询方法，注意，后面没有括号"()"

meth(37) # 现在调用函数
```

上面例子中，meth被称为**绑定方法**。绑定函数是一个可调用对象，它包括了函数（method）和相关的实例。

当调用绑定函数时，相关实例会作为第一个参数传递给函数（self）。

2. 在类上查询

接上面例子：

```
umeth = Foo.instance_method # 在Foo上查询instance_method

umeth(f,37) # 明确提供self，并且调用

```

在这个例子中，umeth被称为未绑定方法。未绑定方法只包含了方法函数，但是需要明确传递正确类型的实例对象作为第一个参数。

在这个例子中，传了Foo的实例f作为第一个参数。

对用户定义的类，绑定方法和未绑定方法都是作为types.MethodType的对象。

下面是方法对象的属性

描述        | 描述           	
------------- | :-------------: 
m.__class__ | 函数定义的类
m.__func__ | 实现方法的函数对象
m.__self__ | 与方法相关的实例，如果没有，返回None

### 内置函数和方法

对象types.BuiltinFunctionType用来描述用c或c++实现的函数和方法。属性和上面类似。

### 类和实例作为可调用对象

类对象和实例也可以作为可调用对象。一个类对象被```class```创建，可以作为一个函数来调用，用来创建类的实例。这种情况下，传递的参数会传递给__init__方法中，用来初始化实例对象。

如果一个实例实现了__call__方法，那么这个实例也可以模拟函数调用。如果对象x定义了__call__函数，那么x(args)会调用x.__call__(args)。

## 类、类型和实例

