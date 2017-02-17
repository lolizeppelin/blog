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
    顺便存在了一个问题,闭包内存是怎么没有释放,这问题我们最后面测试一下

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

注释那一行和 "print new_function" 部分有关,自己可以试试放开注释有什么效果，具体可以参考[这里](http://www.liaoxuefeng.com/wiki/001374738125095c955c1e6d8bb493182103fac9270762a000/001386819879946007bbf6ad052463ab18034f0254bf355000)。

解释下为什么嵌套三层

    第一层, 接受的参数seconds，延迟时间,无此参数可以少一层嵌套
    第二层, 接受的参数的func就是外部函数
    第三层, 接收的参数就是func所接收的参数,func无参数也可以少一层嵌套

如果不用装饰器@delayer(3),那么要实现闭包调用,上述new_function('lalala')就要这样写

```python
# 这里就明白装饰器的实际作用了
# 装饰器只是一个语法糖,简化了套娃一样的写法而已
# 这里具体实现的是闭包, 这种装饰器是函数装饰器
delayer(3)(test)('lalala')
```

现在我们来段opentack的代码,这个装饰器用来限定只调用一起启动。要么用eventlet模式要么用stdlib模式，有点类似我们后面要说的单例模式

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

最后我们再来看看自由变量的释放问题,假设文件topics.py

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

## 写装饰器在结尾

经过学习以后可以明白,装饰器对python来说还是必须要学习的....因为太多大型项目代码这个,你不熟悉就傻逼了

---

### 描述器、类装饰器

接下来我们看描述器,先看完一个很好的翻译文章[link](http://pyzh.readthedocs.io/en/latest/Descriptor-HOW-TO-Guide.html)

看完是不是有点晕...那么我们来慢慢解释下

我们先来描述器的概念

    当一个对象包含有__get__方法,那么这个对象就是一个描述器，例如

```python
class Descriptor(object):
    def __init__(self):
        pass

    def __get__(self, instance, type=None):
        pass
```

上面的Descriptor就是一个描述器的class,下面是一些说明

    这个class必须继承自object
    __get__(self, instance, type=None) 的两个额参数一般必须的,不带这两个的情形我们最后再讨论
    instance 这个参数表示外部对象的实例..后面例子会说明
    type     这个参数表示外部对象的实例的类..后面例子会说明
    上述两个参数，特别是type和关键字相同,所以翻译文档中的参数为
    __get__(self, obj, objtype=None)
    wsgify(openstack实现wsgi的装饰器)里用的是
    __get__(self, instance, owner=None)

    如果一个对象同时定义了 __get__() 和 __set__(),
    它叫做资料描述器(data descriptor)。
    仅定义了 __get__() 的描述器叫非资料描述器(常用于方法，当然其他用途也是可以的)

#### python对象的属性访问过程

    为什么描述器里要讨论这个？因为不明白这个就不知道描述器在干什么

    python默认对属性的访问控制是从对象的字典里面( __dict__ )中
    获取(get),设置(set)和删除(delete)它

    假设一个class的实例为a
    访问a.x 的实际查找顺序：
    先查找 a.__dict__['x']  (我没看过具体的c代码,估计找到这里了也没返回,要去下面的都找一遍确认没有描述器才返回)
    然后查找 type(a).__dict__['x']
    还没找到就查找 type(a) 的父类(不包括元类(metaclass)).
    注：上述查找是通过python内部的__getattribute__()函数实现
        如果__getattribute__()查找属性失败
        会继续调用__getattr__()来找属性
        __getattr__()也没找到的话,才会抛出异常AttributeError

    ---------上面是没有描述器的情况,下面就是有描述器的情况-------

    如果查找到a.x是一个描述器,一般来说是找到的type(a).__dict__['x']是描述器
    Python就会调用描述器的方法来重写默认的控制行为,下面是重写访问行为的过程

对于一个实例a

```python
# 如过 a.x 是描述器, 那么__getattribute__()查找返回可能为
type(a).__dict__['x'].__get__(a, type(a))

# 我们顺便回顾下非描述器的查找顺序
# 如过 a.x 是普通对象, 那么__getattribute__()查找顺序为
# 先找
a.__dict__['x']
# 再找
type(a).__dict__['x']
# 再找
super(type(a)).__dict__['x']
# 如果还没找到,则调用
a.__getattr__(x)
# 等等, a.x是描述器的情况下, a.__dict__['x']要怎么处理？这就要看下面的优先级问题
```

##### 优先级问题,我们拿下面代码来说明

```python
class Descriptor(object):
    def __init__(self):
        pass
    def __get__(self, instance, itype=None):
        return 20
    # def __set__(self, instance, value=None):
    #    pass

class GTest(object):
    def __init__(self, val):
        self.val = val
    val = Descriptor()

a = GTest(10)
print a.val
```

    我们看到class GTest中有属性val
    class GTest的实例a中有属性val
    当我们获取a.val的时候就出现了优先级大问题
    我们的val是
    a.__dict__['x'] (如果class GTest中val不是描述器的话就没有这个优先级问题了)
    还是
    type(a).__dict__['x']

    文档中的优先级是这样的

    于是对于实例来说, 完整的优先级为
    资料料描述器优先于实例变量，实例变量优先于非资料描述器，
    __getattr__() 方法(如果实例中包含的话)具有最低的优先级。

    所以上面代码我们放开set的注释将输出20,没有set输出10


上面我们讨论的对象是实例,对于一个类A, A.x的访问过程

    也是调用A.__getattribute__()
    如果属性是描述器,下面为过程

```python
#__getattribute__ 会把A.x的实际访问变为类似
B.__dict__['x'].__get__(None, B)

# __getattribute__的代码类似

def __getattribute__(self, key):
    "Emulate type_getattro() in Objects/typeobject.c"
    # 这里object是类
    v = object.__getattribute__(self, key)
    if hasattr(v, '__get__'):
       return v.__get__(None, self)
    return v
```

##### function与method

    method(instancemethod)   类中的method是unboundmethod
                             类实例的method是bound method
    function 普通的函数

示例

```python
class GTest(object):
    def __init__(self):
        pass

    def p(self):
        pass
def ll():
    pass
a = GTest()
print ll, type(ll)
print GTest.p, type(GTest.p)
print a.p
```

输出为

```text
<function ll at 0x0000000002A10EB8> <type 'function'>
<unbound method GTest.p> <type 'instancemethod'>
<bound method GTest.p of <__main__.GTest object at 0x0000000002B7D6A0>> <type 'instancemethod'>
```

为什么描述器里说起这个?因为等下的openstack相关代码必须要理解。所以我们要解释下function与method

    1 python中一切都是对象, 所有的函数都有__get__方法,没有__set__方法,所以所有函数(包括方法)都是非资料描述器
    2 method要多一个self参数,function没有
    3 如果一个method变成了function,可以通过__get__方法(因为function是描述器)转为method


经过上面学习以后,我们可以来看openstack用的Wsgifyz的类装饰器是如何工作的了,为了便于理解,删除了部分业务代码

顺便openstack的Wsgify用于处理函数,不会出现类和实例中有同名的属性的情况,所有没有优先级问题。

```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-

class Wsgify(object):

    def __init__(self, func=None, loli=None):
        print '~~~~~~~~~~~~~~Wsgify init'
        self.func = func
        if func is not None:
            print 'init func not none, func is ', func
        else:
            print 'func is none'
        print '~~~~~~~~~~~~~~Wsgify init finsh'

    def __get__(self, instance, owner=None):
        print 'Wsgify __get__'
        if hasattr(self.func, '__get__'):
            print 'instance is', type(instance).__name__, instance.__str__()
            print 'owner is', type(owner).__name__, owner.__name__
            bound_method = self.func.__get__(instance, owner)
            print self.func, bound_method, bound_method.__get__(instance, owner)
            print type(self.func).__name__, type(bound_method).__name__
            return self.clone(self.func.__get__(instance, owner))
        else:
            return self

    def __call__(self, req,  *args, **kwargs):

        func = self.func
        if func is None:
            print 'Wsgify __call__ func none, clone func'
            func = req
            return self.clone(func)
        else:
            print 'Wsgify __call__ func not none, req is ', str(req)
            resp = self.func(req, *args, **kwargs)
            return resp

    def clone(self, func=None, **kw):
        kwargs = {}
        if func is not None:
            kwargs['func'] = func
        return self.__class__(**kwargs)

class Application(object):

    def __init__(self):
        pass

    # wsgi的的app要接收2个参数
    # Application应该写成__call__(self, environ, response)
    # 但是经过一个Wsgify装饰器就变成了只用一个参数
    # Wsgify就是一个描述器类
    # 为了搞清楚Wsgify是如何工作的,就必须把描述器搞清楚
    @Wsgify
    def __call__(self, req):
        print 'app call', req

print '\n-----------------Application init'
a = Application()
print '-----------------Application init finish'
print type(a.__call__)
print '-----------------Application test call finish'

print '\n1st input'
a('test input1')
print '\n2nd input'
a('test input2')
```

输出内容为

```text
~~~~~~~~~~~~~~Wsgify init
init func not none, func is  <function __call__ at 0x0000000002BD5048>
~~~~~~~~~~~~~~Wsgify init finsh

-----------------Application init
-----------------Application init finish
Wsgify __get__
instance is Application <__main__.Application object at 0x0000000002BCD7B8>
owner is type Application
<function __call__ at 0x0000000002BD5048> <bound method Application.__call__ of <__main__.Application object at 0x0000000002BCD7B8>> <bound method Application.__call__ of <__main__.Application object at 0x0000000002BCD7B8>>
function instancemethod
~~~~~~~~~~~~~~Wsgify init
init func not none, func is  <bound method Application.__call__ of <__main__.Application object at 0x0000000002BCD7B8>>
~~~~~~~~~~~~~~Wsgify init finsh
<class '__main__.Wsgify'>
-----------------Application test call finish

1st input
Wsgify __get__
instance is Application <__main__.Application object at 0x0000000002BCD7B8>
owner is type Application
<function __call__ at 0x0000000002BD5048> <bound method Application.__call__ of <__main__.Application object at 0x0000000002BCD7B8>> <bound method Application.__call__ of <__main__.Application object at 0x0000000002BCD7B8>>
function instancemethod
~~~~~~~~~~~~~~Wsgify init
init func not none, func is  <bound method Application.__call__ of <__main__.Application object at 0x0000000002BCD7B8>>
~~~~~~~~~~~~~~Wsgify init finsh
Wsgify __call__ func not none, req is  test input1
app call test input1

2nd input
Wsgify __get__
instance is Application <__main__.Application object at 0x0000000002BCD7B8>
owner is type Application
<function __call__ at 0x0000000002BD5048> <bound method Application.__call__ of <__main__.Application object at 0x0000000002BCD7B8>> <bound method Application.__call__ of <__main__.Application object at 0x0000000002BCD7B8>>
function instancemethod
~~~~~~~~~~~~~~Wsgify init
init func not none, func is  <bound method Application.__call__ of <__main__.Application object at 0x0000000002BCD7B8>>
~~~~~~~~~~~~~~Wsgify init finsh
Wsgify __call__ func not none, req is  test input2
app call test input2
```

我们现在来分析,在一开始,Application的__call__经的装饰过程为Wsgify(__call__),__call__ 变成了而是<class '__main__.Wsgify'>。

Wsgify的实例是一个描述器

我们注意下打印的第二行。

    init func not none, func is  <function __call__ at 0x0000000002BD5048>
    说明在类初始化的时候,传给Wsgify.__init__()的func是function,而不是unboundmethod！！也就是说丢失了self....
    Wsgify.__get__()中的函数就是为了把传入的Application的__call__还原为method

这样后面实例a调用__call__的实际过程为

```python
    type(a).__dict__['__call__'].__get__(a, type(a))
```    
