---
layout: post
title:  "python 闭包.装饰器.描述器.单例模式"
date:   2017-02-15 12:50:00 +0800
categories: "编程"
tag: ["python", "linux"]
---

* content
{:toc}


因为不是专职python开发,不需要多人合作,也没有写过特别大型的代码。标题上的几个功能和概念工作中一直用不上。

所以一直不太想学习相关的内容,因为这些内容太不C了,只是一种额外的写法而已。但是这两天面试的时候别人问了个闭包导致我直接跪了以后,痛定思痛,捡起来算了,反正以后无论是面试还是工作都要用,花1天学习一下也好。

其实装饰器.描述器.单例模式这几个玩意在openstack里有大量应用,我们就直接以openstack的实际代码来说明

#### 闭包.装饰器,这两玩意在一起说是因为这两玩意基本是在一起的

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

我们把上面函数做为装饰器来使用

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

至于count为什么用list不直接用int,可以[参考](http://blog.csdn.net/virtual_func/article/details/50551076),这里的count就是一个函数调用的计数器

我们发现即使没调用任何函数,上述代码也会输出函数test和test2的名字,假设上面的代码在topics.py中


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
```python
1
2
1
2
3
3
4
```

    证明了前面——这个被引用的自由变量将和这个函数一同存在，即使已经离开了创造它的环境也不例外。
    顺便存在了一个问题,闭包内存是怎么释放的

openstack里很少用到count这样的自由变量,这样就不会有上面的自由变量内存不释放的问题了

前面的装饰器比较简单,装饰的函数不能接收参数,我们再来一个常见的装饰器的例子

```python
import functools

def delayer(seconds):
    """
    seconds: 延迟的秒数
    装饰器用于延迟执行函数
    """
    def wrapper(func):
        # @functools.wraps(func)
        def _wrapper(*args, **kwargs):
            _start = time.time()
            time.sleep(seconds)
            ret = func(*args, **kwargs)
            print 'time use:', time.time() - _start
            return ret
        return _wrapper
    return wrapper

@delayer(3)
def test(a):
    print a
    return 'bbb'

new_function = test
print new_function.__name__
new_function('lalala')
```

    注释那一行和 "print new_function.__name__" 部分有关,自己可以试试放开注释有什么效果。

解释下为什么嵌套三层

    第一层, 接受的参数seconds，延迟时间,无此参数可以少一层嵌套
    第二层, 接受的参数的func就是外部函数
    第三层, 接收的参数就是func所接收的参数,func无参数也可以少一层嵌套

如果不用装饰器@delayer(3),那么要实现闭包调用,上述new_function('lalala')就要这样写

```python
# 这里就明白装饰器的实际作用了
# 装饰器是一个语法糖,实际的原理是闭包的工作原理
delayer(3)(test)('lalala')
```

结尾我们来段opentack的代码,这个装饰器用来限定只调用一起启动。要么用eventlet模式要么用stdlib模式，有点类似我们后面要说的单例模式

```python
def configure_once(name):
    """Ensure that environment configuration is only run once.

    If environment is reconfigured in the same way then it is ignored.
    It is an error to attempt to reconfigure environment in a different way.
    """
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            global _configured
            if _configured:
                if _configured == name:
                    return
                else:
                    raise SystemError("Environment has already been "
                                      "configured as %s" % _configured)

            LOG.debug("Environment configured as: %s", name)
            _configured = name
            return func(*args, **kwargs)

        return wrapper

@configure_once('eventlet')
def use_eventlet(monkeypatch_thread=None):
    global httplib, subprocess, Server
    ...

@configure_once('stdlib')
def use_stdlib():
    global httplib, subprocess
    ...


```
## 写在结尾

经过学习以后可以明白,装饰器对python来说还是必须要学习的....因为太多大型项目代码这个,你不熟悉就傻逼了

---
