---
layout: post
title: "python 模块知识"
date: 2016-02-26 19:32:01 +0800
comments: true
categories: 
- python
---

# 模块对象

每一个源文件就是对应一个model。
模块与模块之间的依赖通过import和from语句进行关联。
在其它语言中，有全局变量可供模块之间共享，python中，全局变量在模块间不共享，只属于一个模块。

模块名与模块的文件名是一样的。

模块也是一个对象，因此可以将模块作为参数传递，赋值给变量或存数容器，作为函数结果返回等。是*first-class*对象。

import语句的语法是：
```python
import modname [as varname][,...]
```

这行语句执行完成，会把变量modname绑定到模块对象上。

模块内的语句，在第一次被import时执行。模块对象先于语句执行而创建，并存放在```sys.modules```中。


## 模块属性

import语句会为模块内的所有属性创建一个模块的命名空间（只有第一次import才会执行模块内语句）。模块体内的语句就是在这个命名空间内创建和绑定模块的属性，def创建函数属性，class创建类属性等等。

除了模块体明确定义的属性，在执行模块内语句之前，import还会额外给模块创建一些属性：

比如：

```python

['__all__', '__builtins__', '__doc__', '__file__', '__get_builtin_constructor', '__name__', '__package__',
```

* __dict__ : 用这个字典对象作为模块属性的命名空间。__dict__在模块内不能直接使用，其它（比如__file__，因为是在执行语句之前就创建好了）可以。对于模块M来说，M.S=x 实际上恒等于M.__dict__['S']=x.


比如：

```python
print __file__
print __dict__
```

输出结果：

```
/Users/user/work/project/auto_deploy/tst.py
Traceback (most recent call last):
  File "/Users/user/work/project/auto_deploy/tst.py", line 6, in <module>
    print __dict__
NameError: name '__dict__' is not defined

```
* __name__：模块名
* __file__：模块文件 
* __doc__：注释


## 模块私有化属性

python模块没有真正的私有化属性，按照惯例，只要在模块内定义的属性是以单下划线开头，就认为是私有的，比如_secret.


## 模块加载

实际上，python加载模块是靠内置的```__import__```函数进行的。我们也可以直接调用这个函数。import M，```__import__```会先用key M去查sys.modules，如果在里面，直接返回值；如果不在，```__import__```会创建一个空模块对象，并增加```__name__```属性，其值是M，并把这个模块赋值给 sys.modules[M] ，然后按照正确方式加载模块对象（后面有）。因此只有第一次import一个对象时是慢的，以后直接从sys.modules对象中返回。

### 在磁盘系统搜索模块

当加载一个模块，先检查是否是内置对象（在sys.builtin_module_names 中列出），如果不在，则从磁盘中搜索。

*__import__*会按照sys.path列表中存的路径顺序一个个搜索模块。sys.path中既可以是目录，也可以是zip格式的归档文件。python启动时，会用PYTHONPATH来初始会sys.path。sys.path的第一个字符串总是程序启动的当前目录。


因此，可以通过修改sys.path来操纵搜索逻辑。

### 重新加载模块

可以通过调用reload.reload(M)函数重新加载模块，reload函数的参数是M对象，而不是字符串。通过重新加载模块，影响的是通过import M这种方式，并通过M.A进行访问的代码。那种通过from M import 引入的则不受影响。

### 循环引用

python允许循环引用，比如a.py中import b，而b.py中import a。

已import a为例说明循环引用的加载逻辑

循环引用的执行过程：

import a语句会创建一个空模块对象，并赋值给sys.models['a'] , 
然后开始执行a模块语句，如果遇到import b，则开始加载模块b，同样创建模块对象，并赋值给sys.models['b'].
执行模块b的语句。这是a模块的语句执行是阻塞的，直到b执行完成。

这样，执行模块b的body时，遇到import a，则由于之前已经通过赋值sys.models['a']有数据，因此直接返回这个未初始化完成的a模块对象。这样，有可能a中的属性还没有被初始化。


# 包

python中，一个目录下需要定义```__init__```.py文件，才能标记这个目录是个包。包的体就是```__init__.py```文件内容。

导入包的时候，会首先执行```__init__.py```内容。

## 包的特殊属性

包P的```__file__```的值实际上是包P体的内容，也就是```P/__init__.py```的路径。

比如：

```python
import view
print view.__file__
```
结果是

```
/Users/user/work/project/m-api/view/__init__.pyc
```

可以在*P/__init__.py*中设置*__all__*全局变量，用来控制其他模块执行```from P import *```语句时的行为。如果没有设置这个属性，则```from P import *```不会导入任何P的模块。
























