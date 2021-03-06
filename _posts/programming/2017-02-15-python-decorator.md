---
layout: post
title:  "python 装饰器"
date:   2017-02-15 12:50:00 +0800
categories: "编程"
tag: ["python", "linux"]
---

* content
{:toc}


因为不是专职python开发,不需要多人合作,也没有写过特别大型的代码。装饰器、描述器、闭包的概念工作中一直用不上。

所以一直不太想学习相关的内容,因为这些内容太不C了,只是一种额外的写法而已。但是这两天面试的时候别人问了个闭包导致我直接跪了以后,痛定思痛,捡起来算了,反正以后无论是面试还是工作都要用,花1天学习一下也好。

我先来看装饰器,先来看一个最常见的装饰器写法

```python
def a(func):
    print func.__name__
    def b():
        print 'decorator!!!!!'
        func()
    return b

@a
def test():
    pass

@a
def test2():
    pass
```

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

@delayer(seconds=3)
def test(a):
    print a
    return 'bbb'

new_function = test
print new_function.__name__
new_function('lalala')
```

注释那一行和 "print new_function" 部分有关,自己可以试试放开注释有什么效果，具体可以参考[这里](http://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/001386819879946007bbf6ad052463ab18034f0254bf355000)。

解释下为什么嵌套三层

    第一层, 接受的参数seconds，延迟时间,无此参数可以少一层嵌套
    第二层, 接受的参数的func就是外部函数,也就是需要"装饰"的函数
    第三层, 接收的参数就是func所接收的参数,func无参数也可以少一层嵌套

如果不用装饰器的写法@delayer(3)来实现,那上述new_function('lalala')就要这样写

```python
delayer(3)(test)('lalala')
```

## 这里就明白装饰器的实际作用了,装饰器只是一个语法糖,简化了套娃一样的写法而已

##### 这里的装饰器上的delayer是函数, 这种装饰器就是函数装饰

函数装饰器就会牵扯到闭包这个概念,当delayer中有复杂内容时候,具体参考[python 闭包](http://www.lolizeppelin.com/2017/02/15/python-closure/)

---

##### 如果装饰器不是一个函数而是一个类的话,那么装饰器就是类装饰器

例如

```python
class Descriptor(object):
    def __init__(self, func):
        pass

```

原理都是一样,套娃语法糖,如果需要传入参数,需要给类增加__call__方法

```python
class Descriptor(object):
    def __init__(self, arg):
        pass

    def __call__(self, func)：
        pass

```


类装饰器一般用于装饰类中的方法,原理会涉及到[python 描述器](http://www.lolizeppelin.com/2017/02/15/python-descriptor/)

---

最后我们来段opentack的代码,这个函数装饰器用来限定只调用一起启动。要么用eventlet模式要么用stdlib模式，这里也算是python的单列模式实现方法

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
