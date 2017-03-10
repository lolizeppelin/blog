---
layout: post
title:  "python 闭包"
date:   2017-02-15 12:50:00 +0800
categories: "编程"
tag: ["python", "linux"]
---

* content
{:toc}


#### 闭包定义

    专业的解释:闭包（Closure）是词法闭包（Lexical Closure）的简称，
    它指的是代码块和作用域环境的结合，是引用了自由变量的函数。
    如果在一个内部函数里，对在外部作用域(但不是在全局作用域）的变量进行引用，
    那么内部函数就被认为是闭包（closure）。
    定义在外部函数内的但由内部函数引用或者使用的变量被称为自由变量。闭包在函数式编程中是一个重要的概念。
    这个被引用的自由变量将和这个函数一同存在，即使已经离开了创造它的环境也不例外。

我们来看看一个简单的闭包

```python
def a(func):
    def b():
        func()
    return b

```

一个函数a里面又定义了函数b, 当函数b引用了a作用域中的变量func的时候,b就是一个闭包。func就是自由变量

我们把上面函数做为装饰器来使用,假设文件为topics

```python
def a(func):
    x = [0]
    print func.__name__
    def b():
        x[0] += 1
        print x[0]
        func()
    return b

@a
def test():
    pass

@a
def test2():
    pass
```

至于自由变量count为什么用list不直接用int,可以[参考](http://blog.csdn.net/virtual_func/article/details/50551076),这里的count就是一个函数调用的计数器

直接执行topics.py我们发现即使没调用任何函数,上述代码也会输出函数test和test2的名字

```python
import topics
topics.test()
topics.test()
topics.test2()
topics.test2()
topics.test()
topics.test2()
del topics
import topics
topics.test()
```

输出为

```text
test
test2
1
2
1
2
3
3
4
```

    证明了前面——这个被引用的自由变量将和这个函数一同存在，即使已经离开了创造它的环境也不例外。
    顺便存在了一个问题,闭包内存是怎么没有释放

#### 我们再来看看自由变量的释放问题,假设文件topics.py

```python
def counter(func):
    count = [0]
    def wrapper(*args, **kwargs):
        count[0] += 1
        print 'count: ', str(count[0]),
        return func(*args, **kwargs)
    return wrapper

class Myest(object):
    def __init__(self):
        pass

    @counter
    def t(self, arg='in defuault t'):
        print str(type(self)) + arg
        pass

    @counter
    def u(self, arg='in defuault u'):
        print str(type(self)) + arg

class Myest2(Myest):
    def __init__(self):
        Myest.__init__(self)

    @counter
    def u(self, arg='in defuault u'):
        print str(type(self)) + arg

@counter
def text(arg='out default'):
    print arg

```

假设文件为toptest

```python
from topics import *

def loli():
    text()

def lolita():
    t = Myest2()
    t.u()
```

我们执行如下代码

```python
from topics import *
import toptest
text()
text()
t = Myest()
x = Myest()
t.t()
t.t()
x.t()
t.t()
t.u()
x.u()
x.t()
del t
del x
y = Myest()
y.t()
z = Myest2()
z.t()
z.u()
toptest.loli()
```

输出为

```text
count:  1 out default
count:  2 out default
count:  1 <class '__main__.Myest'>in defuault t
count:  2 <class '__main__.Myest'>in defuault t
count:  3 <class '__main__.Myest'>in defuault t
count:  4 <class '__main__.Myest'>in defuault t
count:  1 <class '__main__.Myest'>in defuault u
count:  2 <class '__main__.Myest'>in defuault u
count:  5 <class '__main__.Myest'>in defuault t
count:  6 <class '__main__.Myest'>in defuault t
count:  7 <class '__main__.Myest2'>in defuault t
count:  1 <class '__main__.Myest2'>in defuault u
count:  3 out default
count:  2 <class 'topics.Myest2'>in defuault u
```

我草泥马好恐怖啊完全不会释放

openstack里很少用到count这样的自由变量,这样就不会有上面的自由变量内存不释放的问题了

要更理解闭包,我们就要先理解另外一个概念——[python的单例模式]()
