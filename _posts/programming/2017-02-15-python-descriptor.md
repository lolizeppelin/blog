---
layout: post
title:  "python 描述器"
date:   2017-02-15 12:50:00 +0800
categories: "编程"
tag: ["python", "linux"]
---

* content
{:toc}


### 描述器,类装饰器

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
    type(a).__dict__['x'].__get__(a, type(a)):

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


经过上面学习以后,我们可以来看openstack用的Wsgify的类装饰器是如何工作的了,如何分发到controler上的.


```python
# 为了便于理解,删除了部分业务代码
# 顺便openstack的Wsgify用于处理函数,不会出现类和实例中有同名的属性的情况,所有没有优先级问题。
# 这个class是我们简化了的Wsgify描述器
class Wsgify(object):
    def __init__(self, func=None, loli=None):
        print '~~~~~~~~~~~~~~Wsgify init'
        self.func = func
        if func is not None:
            print 'init func not none, func is ', func
        else:
            print 'func is none'
        print '~~~~~~~~~~~~~~Wsgify init finsh\n'

    def __get__(self, instance, owner=None):
        print 'Wsgify __get__'
        if hasattr(self.func, '__get__'):
            print 'instance is', type(instance).__name__, instance.__str__()
            print 'owner is', type(owner).__name__, owner.__name__
            bound_method = self.func.__get__(instance, owner)
            print self.func, bound_method, bound_method.__get__(instance, owner)
            print type(self.func).__name__, type(bound_method).__name__
            clone = self.clone(self.func.__get__(instance, owner))
            print 'retrun clone', clone, '\n'
            return clone
        else:
            return self

    def __call__(self, req,  *args, **kwargs):
        func = self.func
        # 这里的作用后面说明
        if func is None:
            print 'Wsgify __call__ self.func none, clone func'
            func = req
            return self.clone(func)
        print 'Wsgify __call__ self.func is', func.__name__, 'with req:', str(req),
        if len(args) > 0:
            print 'args:', args[0],
        print ''
        if isinstance(req, dict):
            if len(args) != 1 or kwargs:
                raise TypeError(
                    "Calling %r as a WSGI app with the wrong signature")
            environ = req
            start_response = args[0]
            # 原代码为req = self.RequestClass(environ)
            # 也就是req是一个RequestClass实例
            # 便于测试我们简化为
            req = 'requets'
            args = ()
            resp_callable = self.func(req, *args, **kwargs)
            # start_response是最外层传入的
            return resp_callable(environ, start_response)
        else:
            return self.func(req, *args, **kwargs)

    def clone(self, func=None, **kw):
        print '\nclone Wsgify intance',
        kwargs = {}
        if func is not None:
            print 'with is function ', func.__name__,
            kwargs['func'] = func
        print ''
        return self.__class__(**kwargs)

class RoutesMiddleware(object):

    def __init__(self, wsgi_app):
        print '*********init an RoutesMiddleware************'
        self.app = wsgi_app

    def __call__(self, environ, start_response):
        print 'RoutesMiddleware call' ,type(self.app),environ, start_response

        response = self.app(environ, start_response)
        return response


class Router(object):

    def __init__(self):
        self._router = RoutesMiddleware(self._dispatch)

    @staticmethod
    @Wsgify
    def _dispatch(req):
        print '_dispatch!!!!!!!!!!!! to contoler'
        return contoler

    @Wsgify
    def __call__(self, req):
        print '===============before return route ==================='
        return self._router

def contoler(environ, start_response):
    print 'contoler says',
    print environ, start_response
    return 'contoler out with' + start_response

print '-----------------Router init'
a = Router()
print '-----------------Router init finish'
print '-----------------Router test call finish'
print '\n++++++++++++++++++++++++++++++++++++++++++++\n'
x = a({'name': '1st input'}, 'wtf')
print x
print '\n++++++++++++++++++++++++++++++++++++++++++++\n'
x = a({'name': '2nd input'}, 'fuck!!!')
print x
```

输出内容为

```text
~~~~~~~~~~~~~~Wsgify init
init func not none, func is  <function _dispatch at 0x0000000002A10128>
~~~~~~~~~~~~~~Wsgify init finsh

~~~~~~~~~~~~~~Wsgify init
init func not none, func is  <function __call__ at 0x0000000002A10198>
~~~~~~~~~~~~~~Wsgify init finsh

-----------------Router init
*********init an RoutesMiddleware************
-----------------Router init finish
-----------------Router test call finish

++++++++++++++++++++++++++++++++++++++++++++

Wsgify __get__
instance is Router <__main__.Router object at 0x0000000002A09AC8>
owner is type Router
<function __call__ at 0x0000000002A10198> <bound method Router.__call__ of <__main__.Router object at 0x0000000002A09AC8>> <bound method Router.__call__ of <__main__.Router object at 0x0000000002A09AC8>>
function instancemethod

clone Wsgify intance with is function  __call__
~~~~~~~~~~~~~~Wsgify init
init func not none, func is  <bound method Router.__call__ of <__main__.Router object at 0x0000000002A09AC8>>
~~~~~~~~~~~~~~Wsgify init finsh

Wsgify __call__ self.func is __call__ with req: {'name': '1st input'} args: wtf
===============before return route ===================
RoutesMiddleware call <class '__main__.Wsgify'> {'name': '1st input'} wtf
Wsgify __call__ self.func is _dispatch with req: {'name': '1st input'} args: wtf
_dispatch!!!!!!!!!!!! to contoler
contoler says {'name': '1st input'} wtf
contoler out withwtf

++++++++++++++++++++++++++++++++++++++++++++

Wsgify __get__
instance is Router <__main__.Router object at 0x0000000002A09AC8>
owner is type Router
<function __call__ at 0x0000000002A10198> <bound method Router.__call__ of <__main__.Router object at 0x0000000002A09AC8>> <bound method Router.__call__ of <__main__.Router object at 0x0000000002A09AC8>>
function instancemethod

clone Wsgify intance with is function  __call__
~~~~~~~~~~~~~~Wsgify init
init func not none, func is  <bound method Router.__call__ of <__main__.Router object at 0x0000000002A09AC8>>
~~~~~~~~~~~~~~Wsgify init finsh

Wsgify __call__ self.func is __call__ with req: {'name': '2nd input'} args: fuck!!!
===============before return route ===================
RoutesMiddleware call <class '__main__.Wsgify'> {'name': '2nd input'} fuck!!!
Wsgify __call__ self.func is _dispatch with req: {'name': '2nd input'} args: fuck!!!
_dispatch!!!!!!!!!!!! to contoler
contoler says {'name': '2nd input'} fuck!!!
contoler out withfuck!!!
```

我们现在来分析,在一开始,Router的__call__经的装饰过程为Wsgify(__call__),__call__ 变成了而是<class '__main__.Wsgify'>

同样的行为作用在_dispatch上


我们注意下打印的第二行。

    init func not none, func is  <function __call__ at 0x0000000002BD5048>
    说明在类初始化的时候,传给Wsgify.__init__()的func是function
    而不是unboundmethod！！也就是说丢失了self....（method变function的说明看本节最末尾）
    Wsgify.__get__()中的函数就是为了把传入的Router的__call__还原为method
    staticmethod的作用是把_dispatch当成外部函数不是类中的函数,可以理解为如下写法

```python
def out_function:
    ...

class Router(object):

    @Wsgify
    def __call__(req):
        ...
    # 用staticmethod只为了简化写法
    _dispatch = Wsgify(out_function)

```

因为__call__属性是描述器,所以这样后面实例a调用__call__的实际过程为

```python
    type(a).__dict__['__call__'].__get__(a, type(a))
```

所以当我们当我们调用

```python
x = a({'name': '1st input'}, 'wtf')
# 也就是
x = a.__call__({'name': '1st input'}, 'wtf')
# 由于Router.__call__是描述器, 那么a.__call__就是
a.__call__ === type(a).__dict__['__call__'].__get__(a, type(a))
# 也就是说完整的函数call为
type(a).__dict__['__call__'].__get__(a, type(a))({'name': '1st input'}, 'wtf')

```

可以看见我们的输出为

```text
Wsgify __get__          # 描述器Wsgify的里的get
instance is Router <__main__.Router object at 0x0000000002A09AC8>   # __get__传入的第一个参数
owner is type Router   # 传入的第二个参数
# 这里就是把function转换回method
<function __call__ at 0x0000000002A10198> <bound method Router.__call__ of <__main__.Router object at 0x0000000002A09AC8>> <bound method Router.__call__ of <__main__.Router object at 0x0000000002A09AC8>>
function instancemethod


clone Wsgify intance with is function  __call__   # 描述器Wsgify调用clone生成一个新的实例,这时候传入的func是instancemethod了
~~~~~~~~~~~~~~Wsgify init
init func not none, func is  <bound method Router.__call__ of <__main__.Router object at 0x0000000002A09AC8>>
~~~~~~~~~~~~~~Wsgify init finsh

retrun clone <__main__.Wsgify object at 0x0000000002869B38>     # 这里就把刚才克隆好的自己返回

```

也就是说,实际的a({'name': '1st input'}, 'wtf')为

    "clone <__main__.Wsgify object at 0x0000000002869B38>".__call__({'name': '1st input'}, 'wtf')

也就是说到现在还没有用到Router原来的call,那么我们来看看最后怎么进去的

```text
Wsgify __call__ self.func is __call__ with req: {'name': '1st input'} args: wtf     # Wsgify.__call__调用了 self.func.__call__, 这
===============before return route ===================  # 终于用到了Router原来的call
RoutesMiddleware call <class '__main__.Wsgify'> {'name': '1st input'} wtf    # RoutesMiddleware的__call__
Wsgify __call__ self.func is _dispatch with req: {'name': '1st input'} args: wtf  # _dispatch是外部function,Wsgify描述器无效,不走__get___,下面有解释
_dispatch!!!!!!!!!!!! to contoler  # Wsgify __call__调用了_dispatch(当参数是dict的时候), _dispatch返回了一个一个函数contoler
contoler says {'name': '1st input'} wtf # 函数contoler 被调用
contoler out withwtf # 函数contoler 返回
```

function的描述器没有走\__get__这个点是个疑惑

```python
def pp(a):
    print a

class WP(object):

    def __init__(self):
        pass

    function = pp

    @staticmethod
    def ppp(a):
        print a


print WP.ppp
print WP.function, id(WP.function)
print pp, id(pp)
a = WP()
a.function()
try:
    a.function('a')
except Exception, e:
    print e
gg = a.ppp.__get__(a, type(a))
print gg
try:
    print gg('lalala')
except Exception,e:
    print e
gg()

```

输出为

```text
<function ppp at 0x0000000002A68EB8>
<unbound method WP.pp> 42723080  # pp被复制了, 不再是原来的pp,唯一的参数相当于self
<function pp at 0x0000000002930EB8> 43191992
<__main__.WP object at 0x0000000002A9D6D8>
pp() takes exactly 1 argument (2 given)
<bound method WP.ppp of <__main__.WP object at 0x0000000002A9D6D8>>
ppp() takes exactly 1 argument (2 given)
<__main__.WP object at 0x0000000002A9D6D8>
```

结论

    当类的属性是一个staticmethod修饰的,也就是外部function的时候
    Wsgify(_dispatch)被看作为普通实例,__get__ 方法不被调用
    那么

```python
# 函数调用
_dispatch(req)
# 相当于调用
Wsgify(_dispatch)(req)
# 也就是
Wsgify.__call__(req)
```

接下来我们解决具体的openstack中代码的问题

1. openstack中@webob.dec.wsgify(RequestClass=Request)是怎么工作的
2. RoutesMiddleware是干什么的
3. \_dispatch 是干什么的
4. 请求是怎么分发到openstack的各个contoler上的

#### 第一个问题好办

```python
class A:
    @Wsgify
    def __call__(self, arg):
        pass

```
    @Wsgify
    根据装饰器的原理,装饰A.__call__过程为
    Wsgify(A.__call__)
    Wsgify有参数的时候
    @Wsgify(loli='xxx')
    根据装饰器的原理,装饰A.__call__过程变为
    Wsgify(loli='xxx')(A.__call__) === Wsgify(loli='xxx').__call__(A.__call__)
    也就是说Wsgify自己也要有__call__函数,也就是说
    A.__call__就走到了Wsgify.__call__中

我们来看看Wsgify的__call__

```python
class Wsgify(object):
    def __init__(self, loli=None, func=None):
        self.loli = loli
        self.func = func

    def __call__(self, req,  *args, **kwargs):
        func = self.func
        if func is None:
            print 'Wsgify __call__ self.func none, clone func'
            func = req
            # 将func传入再克隆自身
            # 这样就达到给Wsgify传其他参数的目的了
            # 稍微多套了一层而已
            return self.clone(func)
```


#### 第二个问题, 原始RoutesMiddleware的\__init__为

```python
class RoutesMiddleware(object):

    def __init__(self, wsgi_app, mapper, use_method_override=True,
                 path_info=True, singleton=True):
        # 这个就是传入的 描述器描述过的function _dispatch
        self.app = wsgi_app
        # 这个mapper就是真正的route,不是我们前面的Route类
        # 是python通用route模块的routes.Mapper类实例
        self.mapper = mapper
        ...
```

    可以看出RoutesMiddleware是用来封装route.mapper(传入的mapper就是routes.Mapper类)
    它只在Router初始化的时候创建一个实例。
    RoutesMiddleware的__call__获取http数据,然后调用mapper分发到contoler

#### 第三个、第四个问题是一起的

    前面不是说分发由RoutesMiddleware做么?第四个问题不是解决了？
    这里为了灵活的处理分发,实际分发的时候传到了外面的_dispatch中
    我们来看看RoutesMiddleware.__call__ 和_dispatch中的具体原始代码

```python

class RoutesMiddleware(object):
    ...

    def __call__(self, environ, start_response):
        ...

        match, route = self.mapper.routematch(environ=environ)
        # 设置 routing_args
        environ['wsgiorg.routing_args'] = ((url), match)
        environ['routes.route'] = route
        environ['routes.url'] = url
        # _dispatch中隐射的controller
        # 这里看出controller能call able,也能接收environ, start_response
        # 也就是说所有controller都是wsgi的app
        response = self.app(environ, start_response)
        ...


class Router(object):
    ....

    @staticmethod
    @webob.dec.wsgify(RequestClass=Request)
    def _dispatch(req):
        # 取出前面设置的routing_args
        match = req.environ['wsgiorg.routing_args'][1]
        if not match:
            return webob.exc.HTTPNotFound()
        # 映射到controller
        app = match['controller']
        return app
```

    看见了把, RoutesMiddleware通过mapper生成match, match塞入environ
    再从_dispatch 从match中取出match, 然后返回app
    这里可以看出其实代码可以完全写得不需要一个_dispatch
    把_dispatch放在外面是为了方便继承重写,修改重定向controller
    顺便,可以看出controller也是一个wsgi app

我们拿osapi_compute_app_legacy_v2来看看mapper如何映射到Controller,同时确认Controller是wsgi app

```python
# api-paste.ini中osapi_compute_app_legacy_v2指向的class
class APIRouter(nova.api.openstack.APIRouter):
    ...

    def _setup_routes(self, mapper, ext_mgr, init_only):
        if init_only is None or 'versions' in init_only:
            self.resources['versions'] = legacy_v2_versions.create_resource()
            mapper.connect("versions", "/",
                        # 所以具体的controller就是legacy_v2_versions.create_resource()
                        controller=self.resources['versions'],
                        action='show',
                        conditions={"method": ['GET']})
```

```python
# 找到nova.api.openstack.APIRouter中
import routes
class APIMapper(routes.Mapper):
    def routematch(self, url=None, environ=None):
        ...

    def connect(self, *args, **kargs):
        ...

class ProjectMapper(APIMapper):
    def resource(self, member_name, collection_name, **kwargs):
        ...

# nova.api.openstack.APIRouter
class APIRouter(base_wsgi.Router):  # base_wsgi.Router就是我们前面实例的Router
    @classmethod
    def factory(cls, global_config, **local_config):
        """Simple paste factory, :class:`nova.wsgi.Router` doesn't have one."""
        return cls()

    def __init__(self, ext_mgr=None, init_only=None):
        ...
        mapper = ProjectMapper()
        self.resources = {}
        # 这里设置路由,看前面
        self._setup_routes(mapper, ext_mgr, init_only)
```

```python
# 我们从_setup_routes随便找一个mapper.connect看看
# legacy_v2_versions.create_resource()

def create_resource():
    return wsgi.Resource(Controller())

# wsgi.Resource
class Resource(wsgi.Application):
    .....
    # 这里可以证实了所有的Controller都是 wsgi app
    @webob.dec.wsgify(RequestClass=Request)
    def __call__(self, request)
       ....
```


最末尾来看看method变function的过程


```python
class T(object):
    def __init__(self, func):
        print 'init argv func is', type(func)

class A(object):
    def __init__(self):
        pass
    def ppp(self, x):
        print x
    # @T来修饰ppp的话就相当于
    # ppp = T(ppp)
    # 这里专门换成了xx便于理解
    xx = T(ppp)
    def gg(self):
        print 'self.ppp', type(self.ppp)
        print 'self.xx', type(self.xx)
a = A()
a.gg()
```

输出为

  init argv func is <type 'function'>
  self.ppp <type 'instancemethod'>
  self.xx <class '__main__.T'>
