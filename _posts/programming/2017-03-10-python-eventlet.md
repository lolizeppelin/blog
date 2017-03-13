---
layout: post
title:  "python eventlet使用,协程原理"
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
    # 这个就是返回当前正在调用的greenlet类实例
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
        主要属性parent
        parent一般用于存放运行main loop的绿色线程
        main loop的绿色线程的parent直接=self
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

虽然openstack里调用的都是GreenPool和GreenThread

但eventlet的核心的工作是在hub里的,需要熟悉hub后再继续,参考[eventlet中Hub的工作原理]()

看完Hub的原理我们来看看L3 Agent的入口start部分,找到绿色线程的入口代码

```python
server = neutron_service.Service.create(
    binary='neutron-l3-agent',
    topic=topics.L3_AGENT,
    report_interval=cfg.CONF.AGENT.report_interval,
    manager=manager)

class Service(n_rpc.Service):
    ...

    def start(self):
        ...
        # 省略非关键代码
        #  n_rpc.Service中的代码合并过来
        super(Service, self).start()
        self.conn = create_connection()
        LOG.debug("Creating Consumer connection for Service %s",
                  self.topic)
        # 关键点!!!!
        # self.manager是L3NATAgent
        # L3NATAgent作为endpoints传到rpc接口        
        endpoints = [self.manager]
        self.conn.create_consumer(self.topic, endpoints)
        # Hook to allow the manager to do other initializations after
        # the rpc connection is created.
        if callable(getattr(self.manager, 'initialize_service_hook', None)):
            self.manager.initialize_service_hook(self)
        # Consume from all consumers in threads
        self.conn.consume_in_threads()
        self.manager.after_start()
        ....


class L3NATAgent:

    def after_start(self):
        .....
        eventlet.spawn_n(self._process_routers_loop)
        LOG.info(_LI("L3 agent started"))

    def _process_routers_loop(self):
        ...
        pool = eventlet.GreenPool(size=8)
        while True:
            pool.spawn_n(self._process_router_update)

```

最初的入口是eventlet.spawn_n,我们来看看

```python
# 简单看看GreenThread

class GreenThread(greenlet.greenlet):
    # 继承自greenlet
    # 少了func属性
    # 将自身main函数互作为func来初始化greenlet
    def __init__(self, parent):
        greenlet.greenlet.__init__(self, self.main, parent)
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

# 下面是各个spawn

def spawn(func, *args, **kwargs):   # 这个相当seconds为0的spawn_after
    hub = hubs.get_hub()
    # 生成绿色线程
    g = GreenThread(hub.greenlet)
    # 将这个绿色线程注册到hub
    hub.schedule_call_global(0, g.switch, func, args, kwargs)
    return g

def spawn_n(func, *args, **kwargs): # 这个就是被agent调用的,他和spawn有什么区别呢?
    return _spawn_n(0, func, args, kwargs)[1]

def _spawn_n(seconds, func, args, kwargs):
    hub = hubs.get_hub()
    # 生成绿色线程
    g = greenlet.greenlet(func, parent=hub.greenlet)
    # 将这个绿色线程注册到hub
    t = hub.schedule_call_global(seconds, g.switch, *args, **kwargs)
    return t, g

def spawn_after(seconds, func, *args, **kwargs):    # 这个相当于可以带seconds参数的spawn
    hub = hubs.get_hub()
    g = GreenThread(hub.greenlet)
    hub.schedule_call_global(seconds, g.switch, func, args, kwargs)
    return g

```
