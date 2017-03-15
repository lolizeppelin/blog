---
layout: post
title:  "python eventlet使用,绿色线程工作原理"
date:   2017-03-10 12:50:00 +0800
categories: "编程"
tag: ["python", "linux"]
---

* content
{:toc}


eventlet基于greenlet,greenlet是c写的,但是我放弃看greenlet源码了

因为好奇搜了下greenlet切换和goto什么关系[参考](https://www.zhihu.com/question/23923904/answer/31162137)

    goto没有发生栈帧切换，而greenlet的switch则是更复杂的C栈帧的保存和恢复。
    基本原理是会把%ebp %esp %ebx等寄存器代表的栈帧状态保存到内存本地变量，
    然后把当前greenlet的栈（寄存器的状态已保存到本地变量里）再拷贝到操作系统堆上，
    然后再把要恢复的greenlet的栈帧（之前切出来的时候保存到堆上了）恢复
    （恢复相应寄存器即可，cpu会自动根据寄存器的值继续从上次的指令处执行），
    而每一个greenlet又都有一个父greenlet,所以当当前greenlet执行完毕，
    回到父greenlet即可。C栈切换完后，Python的栈帧切换则更简单一点，
    直接修改PyThreadState的top_frame(可以理解为起到%eip的作用)即可，
    frame是Python运行时的栈帧抽象，记录了指令、符号表、常量表等运行时状态。所以两个不是一回事儿。

所以..放弃阅读greenlet源码直接看greenlet的使用,如果有兴趣读greenlet

可以[参考1](http://www.tuicool.com/articles/2uQjEn), [参考2](http://www.jianshu.com/p/cd41c14b19f4)


greenlet有如下内容

```yaml
CLASSES:
    # GreenletExit异常
    # 这个特定的异常不会波及到父greenlet，它用于干掉一个greenlet。
    - class GreenletExit(exceptions.BaseException)
    - class error(exceptions.Exception)
    # 这个就是greenlet主要类
    - class greenlet
FUNCTIONS:
    # 这个就是返回当前的greenlet类实例
    # 其实greenlet的关键原理和yeid应该是差不多的
    # 这个函数也就是为什么要专门弄一个greenlet而不用yeid的原因
    # yeid协程的互相切换对代码改动太多
    - getcurrent(...)   
    - gettrace(...)
    - settrace(...)
DATA:
    - GREENLET_USE_GC = True
    - GREENLET_USE_TRACING = True
```

我们来详细看看greenlet类的使用代码,这里只是python的调用代码,具体实现在c中

```python

class greenlet(__builtin__.object)
 |  greenlet(run=None, parent=None) -> greenlet
 |  
 |  Creates a new greenlet object (without running it).
 |  
 |   - *run* -- The callable to invoke.
 |   - *parent* -- The parent greenlet. The default is the current greenlet.
 |  
 |  Methods defined here:
 |  
 |  __getstate__(...)
 |  
 |  __init__(...)
 |      x.__init__(...) initializes x; see x.__class__.__doc__ for signature
        主要属性parent, run
        parent一般用于存放运行main loop的绿色线程
        main loop的绿色线程的parent直接=self
        run就是传入的func
 |  
 |  __nonzero__(...)
 |      x.__nonzero__() <==> x != 0
 |  
 |  switch(...)
 |      switch(*args, **kwargs)
 |      
 |      Switch execution to this greenlet.
 |      切换执行的greenlet为当前greenlet
 |      If this greenlet has never been run, then this greenlet
 |      will be switched to using the body of self.run(*args, **kwargs).
        如果这个greenlet还没有调用过run,则会直接调用self.run(*args, **kwargs).

 |      
 |      If the greenlet is active (has been run, but was switch()'ed
 |      out before leaving its run function), then this greenlet will
 |      be resumed and the return value to its switch call will be
 |      None if no arguments are given, the given argument if one
 |      argument is given, or the args tuple and keyword args dict if
 |      multiple arguments are given.
        如果这个greenlet是激活的(调用过run, 然后被switch()到,run也没有结束)
        这个greenlet将被恢复,如果没有参数输入,switch的返回为None
        如果传入参数为*args,返回tuple
        如果传入参数为**kwargs,返回字典

 |      
 |      If the greenlet is dead, or is the current greenlet then this
 |      function will simply return the arguments using the same rules as
 |      above.
        如果switch到的greenlet已经dead或者switch的greenlet就是当前greenlet
        switch将按照前面的规则输出返回值

        这个switch就相当于yeid,传参就相当于send,无参就是next
        返回值相当于yeid返回
 |  
 |  throw(...)
 |      Switches execution to the greenlet ``g``, but immediately raises the
 |      given exception in ``g``.  If no argument is provided, the exception
 |      defaults to ``greenlet.GreenletExit``.  The normal exception
 |      propagation rules apply, as described above.  Note that calling this
 |      method is almost equivalent to the following::
 |      
 |          def raiser():
 |              raise typ, val, tb
 |          g_raiser = greenlet(raiser, parent=g)
 |          g_raiser.switch()
 |      
 |      except that this trick does not work for the
 |      ``greenlet.GreenletExit`` exception, which would not propagate
 |      from ``g_raiser`` to ``g``.
 |  
 |  ----------------------------------------------------------------------
 |  Data descriptors defined here:
 |  
 |  __dict__
 |  
    # 当前greenlet的状态
 |  dead
 |  
    # 这个再看
 |  gr_frame
 |  
    # 父greenlet
 |  parent
 |  
    # 具体的执行函数
 |  run
```

简单的greenlet原理看完以后,就要比较深入的学习eventlet的实现了

虽然openstack里调用的都是GreenPool和GreenThread,但eventlet的核心的工作是在hub里的

需要熟悉hub后再继续,参考[eventlet中Hub的工作原理](http://www.lolizeppelin.com/2017/03/13/python-eventlet-hub/)

我们先看看GreenThread类,它重新封装了greenlet.greenlet

```python

class GreenThread(greenlet.greenlet):
    # 继承自greenlet
    # 少了func属性
    # 将自身main函数互作为func来初始化greenlet
    def __init__(self, parent):
        greenlet.greenlet.__init__(self, self.main, parent)
        # event也是比较复杂的,用原生的greenlet就没有这个Event
        self._exit_event = event.Event()
        self._resolving_links = False

    def main(self, function, args, kwargs):
        # main中才调用func,外部function在main调用的时候才被调用
        try:
            result = function(*args, **kwargs)
        except:
            self._exit_event.send_exception(*sys.exc_info())
            self._resolve_links()
            raise
        else:
            self._exit_event.send(result)
            self._resolve_links()
```

一般我们通过spawn来生成GreenThread,下面是各个spawn


```python

def spawn(func, *args, **kwargs):   # 这个相当seconds为0的spawn_after
    hub = hubs.get_hub()
    # 生成绿色线程GreenThread
    g = GreenThread(hub.greenlet)
    # 将这个绿色封装为timer
    # 然后加入到预备定时器列表中
    hub.schedule_call_global(0, g.switch, func, args, kwargs)
    # 当定时器被hub执行的时候,通过timer的__call__函数
    # 调用g.switch(func, args, kwargs)
    # 因为greenlet.run也就是GreenThread.main还没有被调用过
    # 开始执行GreenThread.main
    # 最终执行func
    # 由于GreenThread的main中还封装了相关的_resolve_links和_exit_event
    # 所以spawn比spawn_n慢
    # 只有一个返回值,不返回timer,只返回绿色线程GreenThread实例
    return g

def spawn_n(func, *args, **kwargs): # 这个就是被agent调用的,我们看看他和spawn有什么区别
    # 也返回绿色线程greenlet.greenlet实例
    return _spawn_n(0, func, args, kwargs)[1]

def _spawn_n(seconds, func, args, kwargs):
    hub = hubs.get_hub()
    # 生成绿色线程greenlet.greenlet
    g = greenlet.greenlet(func, parent=hub.greenlet)
    # 当定时器被hub执行的时候,通过timer的__call__函数
    # 调用g.switch(args, kwargs)
    # 因为greenlet.run也就是func还没有被调用过
    # 开始执行执行func(*args, **kwargs)
    t = hub.schedule_call_global(seconds, g.switch, *args, **kwargs)
    return t, g

def spawn_after(seconds, func, *args, **kwargs):    # 这个相当于可以带seconds参数的spawn
    hub = hubs.get_hub()
    g = GreenThread(hub.greenlet)
    # 带了参数,定时器会被延迟触发
    hub.schedule_call_global(seconds, g.switch, func, args, kwargs)
    return g

# 回顾下Hub.schedule_call_global

class Hub()
    ...
    def schedule_call_global(self, seconds, cb, *args, **kw):
        # 生成定时器类
        t = timer.Timer(seconds, cb, *args, **kw)
        # 这里会添加到预备定时器列表
        self.add_timer(t)
        # 返回定时器实例
        return t

```

我们来详细看看GreenThread里的其他方法

```python
class GreenThread(greenlet.greenlet):
    def __init__(self, parent):
        greenlet.greenlet.__init__(self, self.main, parent)
        # GreenThread区别与greenlet在于
        # 1.有一个_exit_event属性是Event类
        # 2.封装了greenlet的run
        # 在run的前后进行了一些link和unlink
        self._exit_event = event.Event()
        self._resolving_links = False

    def main(self, function, args, kwargs):
        try:
            result = function(*args, **kwargs)
        except:
            # function执行报错,调用send_exception让绿色线程throw出异常
            self._exit_event.send_exception(*sys.exc_info())
            self._resolve_links()
            raise
        else:
            # function执行成功,调用send
            # send最终通过switch接收result
            self._exit_event.send(result)
            self._resolve_links()

    def wait(self):
        return self._exit_event.wait()

    def link(self, func, *curried_args, **curried_kwargs):
        # 如果没有_exit_func，生成退出函数队列
        # 默认为deque对列
        self._exit_func = getattr(self, '_exit_funcs', deque())
        # 把要执行的函数插到self._exit_func队列里
        self._exit_funcs.append((func, curried_args, curried_kwargs))
        # 如果event已经不是NOT_USED标记
        # 也就是说event调用过send之类的函数
        if self._exit_event.ready():
            # 看后面_resolve_links
            self._resolve_links()

    def unlink(self, func, *curried_args, **curried_kwargs):
        # 移除退出函数
        if not getattr(self, '_exit_funcs', None):
            return False
        try:
            self._exit_funcs.remove((func, curried_args, curried_kwargs))
            return True
        except ValueError:
            return False

    def _resolve_links(self):
        # 判断是否已经resolving_links
        if self._resolving_links:
            return
        # 设置resolving_links标记
        # 退出函数也可能是绿色线程会switch到main loop后变到其他绿色线程
        # 并在其他线程里调用_resolve_links
        self._resolving_links = True
        try:
            exit_funcs = getattr(self, '_exit_funcs', deque())
            # 从self._exit_funcs队列中取出所有退出需要执行的函数
            while exit_funcs:
                f, ca, ckw = exit_funcs.popleft()
                # 执行所有退出函数
                f(self, *ca, **ckw)
        finally:
            # 重新设置_resolving_links
            self._resolving_links = False

```
