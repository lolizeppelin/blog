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

至于自由变量count为什么用list不直接用int,在末尾有说明,目前只有只要理解这里的count就是一个函数调用的计数器

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

要更理解闭包和自由变量的内存释放,我们就要先理解另外一个概念——[python的单例模式](http://www.lolizeppelin.com/2017/03/06/python-singleton/)

看完上面单例模式以后,我们来总结下

    在topics.py中的text函数,当被counter修饰以后,生成了一个闭包对象
    我们引用函数的text不再是text函数本身,而是counter返回的闭包函数
    在python中,一切皆对象,class的实例是对象、class本身也是对象,上面的text函数是对象,装饰后的闭包函数也是对象
    通过单模式一节可以知道,其实class相当与一个全局单例对象,text函数和被装饰后的闭包函数也是全局单例对象
    因为这些单例对象模块中,在第一次import生成后就不会消失(除非显式的del掉)
    因为闭包对象一直存在...自然闭包引用的自由变量也不会消失
    自由变量有点类似类似属性

闭包本质上是属于函数式编程的写法(函数生成函数),上面的例子单例的闭包,实际中我们还有很多非单例的闭包,有一些注意事项需要说明,[参考](http://www.jb51.net/article/108916.htm)

1、延迟绑定

```python
def multipliers():
    return [lambda x: i * x for i in range(4)]
print [m(2) for m in multipliers()]
```

上面的输出不是[0, 2, 4, 6]而是[6, 6, 6, 6]. 原因就在于延迟绑定

    multipliers一共生成了4个闭包,按理说自由变量分别是0、1、2、3
    但是python在闭包被调用的时候,自由变量的值才确定
    multipliers执行完毕的时候i约家变成3,所以调用闭包的时候,自由变量绑定的值都是3
    这个问题会同样会出现在c#中,erlang这种专门的函数式编程语言就没有这个问题

消除延迟绑定有两种方法

```python
def multipliers():
    # 传参的方式, 通过参数直接传值到内部    
    return [lambda x, j=i: j* x for i in range(4)]


def multipliers():
    # 通过functools.partial构造偏函数, 使得自由变量优先绑定到闭包函数上
    # functools.partial的作用是把函数a和参数构造成函数b,这样函数b就不需要有参数传入了
    return [functools.partial(lambda i, x: x * i, i) for i in range(4)]
```


2、禁止在闭包函数内对引用的自由变量进行重新绑定,这个可以[参考](http://blog.csdn.net/virtual_func/article/details/50551076)

    前面自由变量count为什么用list不直接用int就是这个原因,不知道算不是python的bug
    有点类似基本类型按值传递的逻辑

上述由multipliers生成的闭包如果不再被引用的话,gc将自动回收对应对象,闭包所引用的自由变量自然也会被回收掉
