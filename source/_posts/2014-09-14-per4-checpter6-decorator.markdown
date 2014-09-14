---
layout: post
title: "python decorator 深入理解"
date: 2014-09-14 11:38:09 +0800
comments: true
categories: 
- python
- decorator
---

# 装饰器介绍

装饰器是一个函数，其主要目的是包装另一个函数或类，来透明的修改或者增强被包装对象的功能。语法上，装饰器用*@*表示，比如：

```python

@trace
def square(x):
	return x*x

```

上面装饰器，类似于这样的：

```python

def square(x):
	return x*x

square = trace(square)

```
<!-- more -->
# 不带参数的装饰器

装饰器不带参数，被装饰的函数可以带参数，也可以不带，

## 被装饰函数不带参数

不带参数的装饰器，比较简单，只需要接收一个函数作为参数即可。
例子：

```python

import time

def timing_function(some_function):  # 装饰器，接收一个函数作为参数

    """
    Outputs the time a function takes
    to execute.
    """

    def wrapper():
        t1 = time.time()
        some_function()
        t2 = time.time()
        return "Time it took to run the function: " + str((t2-t1)) + "\n"
    return wrapper

@timing_function
def my_function():
    num_list = []
    for x in (range(0,10000)):
        num_list.append(x)
    print "\nSum of all the numbers: " +str((sum(num_list)))


print my_function()

```

## 被装饰函数带参数

如果被装饰的函数带参数，让包装函数带着参数就可以。
例子：

```python

# It’s not black magic, you just have to let the wrapper 
# pass the argument:

def a_decorator_passing_arguments(function_to_decorate):
    def a_wrapper_accepting_arguments(arg1, arg2):
        print "I got args! Look:", arg1, arg2
        function_to_decorate(arg1, arg2)
    return a_wrapper_accepting_arguments

# Since when you are calling the function returned by the decorator, you are
# calling the wrapper, passing arguments to the wrapper will let it pass them to 
# the decorated function

@a_decorator_passing_arguments
def print_full_name(first_name, last_name):
    print "My name is", first_name, last_name

print_full_name("Peter", "Venkman")
# outputs:
#I got args! Look: Peter Venkman
#My name is Peter Venkman

```

为了函数通用，可以让包装器接收参数设置为(*args, **kwargs)的形式：

```python

def common_decorator(function):

    """
    Limits how fast the function is
    called.
    """

    def wrapper(*args, **kwargs):
        # 处理代码
        function(*args, **kwargs)
        # 处理代码
    return wrapper


@common_decorator
def print_number(num):
    return num

```

# 装饰器需要参数

有时候为了装饰器的功能性，需要装饰器本身也需要接收参数，但是装饰器应该接收一个函数作为参数，为了达到让装饰器也能接收参数，需要：

* 多套一层函数，最外层函数接收装饰器用到的参数
* 在打装饰器时，把参数传给装饰器，也就是在*@*后面是一个函数调用，而不是仅仅是装饰器的名字。

例子：

```python

def decorator_maker_with_arguments(decorator_arg1, decorator_arg2):

    print "I make decorators! And I accept arguments:", decorator_arg1, decorator_arg2

    def my_decorator(func):
        # The ability to pass arguments here is a gift from closures.
        # If you are not comfortable with closures, you can assume it’s ok,
        # or read: http://stackoverflow.com/questions/13857/can-you-explain-closures-as-they-relate-to-python
        print "I am the decorator. Somehow you passed me arguments:", decorator_arg1, decorator_arg2

        # Don't confuse decorator arguments and function arguments!
        def wrapped(function_arg1, function_arg2) :
            print ("I am the wrapper around the decorated function.\n"
                  "I can access all the variables\n"
                  "\t- from the decorator: {0} {1}\n"
                  "\t- from the function call: {2} {3}\n"
                  "Then I can pass them to the decorated function"
                  .format(decorator_arg1, decorator_arg2,
                          function_arg1, function_arg2))
            return func(function_arg1, function_arg2)

        return wrapped

    return my_decorator

@decorator_maker_with_arguments("Leonard", "Sheldon")
def decorated_function_with_arguments(function_arg1, function_arg2):
    print ("I am the decorated function and only knows about my arguments: {0}"
           " {1}".format(function_arg1, function_arg2))

decorated_function_with_arguments("Rajesh", "Howard")
#outputs:
#I make decorators! And I accept arguments: Leonard Sheldon
#I am the decorator. Somehow you passed me arguments: Leonard Sheldon
#I am the wrapper around the decorated function. 
#I can access all the variables 
#   - from the decorator: Leonard Sheldon 
#   - from the function call: Rajesh Howard 
#Then I can pass them to the decorated function
#I am the decorated function and only knows about my arguments: Rajesh Howard

```

注意上面的例子：
1、打装饰器的时候，其实是一个函数调用：@decorator_maker_with_arguments("Leonard", "Sheldon")。
2、装饰器返回的包装函数，实际上是一个闭包，它引用了装饰器的参数（decorator_arg1和decorator_arg2）。

# functools模块用于装饰器

functools在python2.5引入的，他包含了函数functools.wraps()，这个函数的作用是拷贝被包装的函数的名字、模块和docstring到它的包装器上。
functools。wraps()实际上也是个装饰器。

例子：

```python

# For debugging, the stacktrace prints you the function __name__
def foo():
    print "foo"

print foo.__name__
#outputs: foo

# With a decorator, it gets messy    
def bar(func):
    def wrapper():
        print "bar"
        return func()
    return wrapper

@bar
def foo():
    print "foo"

print foo.__name__
#outputs: wrapper
# 输出的信息是包装器的信息，而不是原始函数的信息。

# 使用"functools" 来解决上面问题

import functools

def bar(func):
    # We say that "wrapper", is wrapping "func"
    # and the magic begins
    @functools.wraps(func)
    def wrapper():
        print "bar"
        return func()
    return wrapper

@bar
def foo():
    print "foo"

print foo.__name__
#outputs: foo
# 输出的信息是实际的foo函数的信息
```

# 类作为装饰器

装饰器除了使用函数的方式外（大部分都是用函数来实现装饰器），还可以使用类的形式。

对于装饰器来说，唯一的约束是：*装饰器返回的对象必须可以被当成函数使用，也就是它可以被调用*。

如果类被当成装饰器，那么，类必须实现__call__函数。

需要注意的点：

* 类的初始化函数（__init__）需要接受一个函数作为参赛
* 在给函数打装饰器时，__init__会执行
* 在调用被装饰的函数时，类的__call__被调用

例子：

```python

>>> class myDecorator(object):
...     def __init__(self, f):
...             print "inside myDecorator.__init__()"
...             f()
...     def __call__(self):
...             print "inside myDecorator.__call__()"
...
>>> @myDecorator
... def aFunction():
...     print "inside aFunction()"
...
inside myDecorator.__init__()
inside aFunction()
>>> aFunction()
inside myDecorator.__call__()

```

## 被装饰函数带参数

如果使用类来作为装饰器，如果被装饰函数需要参数，则定义在__call__函数上。比如：

```python

class decoratorWithoutArguments(object):

    def __init__(self, f):
        """
        If there are no decorator arguments, the function
        to be decorated is passed to the constructor.
        """
        print "Inside __init__()"
        self.f = f

    def __call__(self, *args):  # 传给被装饰器的参数，传递给__call__()
        """
        The __call__ method is not called until the
        decorated function is called.
        """
        print "Inside __call__()"
        self.f(*args)
        print "After self.f(*args)"

@decoratorWithoutArguments
def sayHello(a1, a2, a3, a4):
    print 'sayHello arguments:', a1, a2, a3, a4

print "After decoration"

print "Preparing to call sayHello()"
sayHello("say", "hello", "argument", "list")
print "After first sayHello() call"
sayHello("a", "different", "set of", "arguments")
print "After second sayHello() call"

```

结果是：

```text

# 函数定义的执行结果
Inside __init__()
After decoration

Preparing to call sayHello()  

# 下面结果是一次函数调用
Inside __call__()  
sayHello arguments: say hello argument list
After self.f(*args)

After first sayHello() call

# 下面结果是一次函数调用
Inside __call__()
sayHello arguments: a different set of arguments
After self.f(*args)

After second sayHello() call

```

## 装饰器带参数

如果类作为装饰器，装饰器如果带参数，则需要注意的比较多：

* 参数传递给__init__函数。
* __call__函数需要返回一个包装函数（因为装饰器有参数，所有打装饰器的地方，实际上是个函数调用，这会导致class的__call__调用）。

比如：

```python

class decoratorWithArguments(object):

    def __init__(self, arg1, arg2, arg3):
        """
        If there are decorator arguments, the function
        to be decorated is not passed to the constructor!
        """
        print "Inside __init__()"
        self.arg1 = arg1
        self.arg2 = arg2
        self.arg3 = arg3

    def __call__(self, f):
        """
        If there are decorator arguments, __call__() is only called
        once, as part of the decoration process! You can only give
        it a single argument, which is the function object.
        """
        print "Inside __call__()"
        def wrapped_f(*args):
            print "Inside wrapped_f()"
            print "Decorator arguments:", self.arg1, self.arg2, self.arg3
            f(*args)
            print "After f(*args)"
        return wrapped_f

@decoratorWithArguments("hello", "world", 42)
def sayHello(a1, a2, a3, a4):
    print 'sayHello arguments:', a1, a2, a3, a4

print "After decoration"

print "Preparing to call sayHello()"
sayHello("say", "hello", "argument", "list")
print "after first sayHello() call"
sayHello("a", "different", "set of", "arguments")
print "after second sayHello() call"

```

输出结果是：

```text
定义输出
Inside __init__()
Inside __call__()  # 因此装饰器是函数调用，因此走到__call__调用


After decoration

Preparing to call sayHello()

# 下面结果是一次函数调用装饰器调用
Inside wrapped_f()
Decorator arguments: hello world 42
sayHello arguments: say hello argument list
After f(*args)

after first sayHello() call

# 下面结果是一次函数调用
Inside wrapped_f()
Decorator arguments: hello world 42
sayHello arguments: a different set of arguments
After f(*args)

after second sayHello() call
```

# 总结

装饰器可以由函数实现，也可以由类实现，由类实现需要类实现__call__函数。
不管用哪种方式，装饰器返回的必须是一个接受一个函数参数的可调用对象。

# 实际例子

关于装饰器的实际例子，可以参考

https://wiki.python.org/moin/PythonDecoratorLibrary

# 参考

http://stackoverflow.com/questions/739654/how-can-i-make-a-chain-of-function-decorators-in-python/1594484#1594484

http://www.artima.com/weblogs/viewpost.jsp?thread=240845
http://www.artima.com/weblogs/viewpost.jsp?thread=240808

















