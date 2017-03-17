---
layout: post
title:  "python eventlet中Hub的工作原理"
date:   2017-03-13 12:50:00 +0800
categories: "编程"
tag: ["python", "linux"]
---

* content
{:toc}



我们先来看看hub怎么来的

```python

# 线程局部变量
_threadlocal = threading.local()

def get_hub():
    try:
        # 从线程局部变量中获取hub
        hub = _threadlocal.hub
    except AttributeError:
        try:
            # 这里是判断_threadlocal中是否已经有了Hub函数
            _threadlocal.Hub
        except AttributeError:
            # use_hub内部是根据系统、python库来选择使用的Hub类
            # 在linux中,use_hub()默认为
            #  _threadlocal.Hub = eventlet.hub.epoll.Hub
            # use_hub()里怎么绕到eventlet.hub.epoll.Hub就不再详细看了
            use_hub()
        # 调用_threadlocal.Hub()给_threadlocal.hub赋值
        hub = _threadlocal.hub = _threadlocal.Hub()
    # 上面的代码绕了一下的是为的是实现返回的hub在单个线程中只创建一次的（线程中的单例）
    return hub
```


看Hub类之前要看hub用到的一个类timer,这个类比较重要


```python

class Timer(object):
    # timer就是一个定时器类
    # Hub的main loop里会不停的扫描定时器列表
    # 然后处理里面的timer类实例
    # timer类实例在tpl变量中保存切换绿色线程的函数
    def __init__(self, seconds, cb, *args, **kw):
        # timer被调用的时间
        # 如果要立即被调用,那么这个时间一般是
        # time.time() + 0
        # 如果要延迟0.01秒调用
        # 这个时间是 time.time() + 0.01
        self.seconds = seconds

        # cb这个参数一般是GreenThread/greenlet的switch
        # *args, **kw自然就是传给switch的参数

        # cb也不一定是switch,也可能是event之类用于通知的调用接口
        # 注意这里所说的event不是epoll的event,是eventlet.event.Event
        # 如果是event,里面会有更复杂的绿色线程切换

        # cb也可以直接是一个外部函数
        # 但如果是一个外部函数,这个函数必须是非阻塞的而且运行时间短
        # 否则会影响整个main loop

        # 所以这个cb尽量不要传函数而是传绿色线程的switch或event
        self.tpl = cb, args, kw
        # 当前定时器是否被调用过
        self.called = False
        if _g_debug:
            self.traceback = six.StringIO()
            traceback.print_stack(file=self.traceback)

    @property
    def pending(self):
        return not self.called

    def copy(self):
        cb, args, kw = self.tpl
        return self.__class__(self.seconds, cb, *args, **kw)

    def schedule(self):
        """Schedule this timer to run in the current runloop.
        """
        self.called = False
        self.scheduled_time = get_hub().add_timer(self)
        return self

    def __call__(self, *args):
        # 当这个定时器被调用的时候
        # 激活这个timer中的绿色线程/函数
        # 这个__call__接受的args参数是没用的
        # 这里的参数是防止外部写错了传参导致报错
        # 所以timer的调用都是直接timer()
        if not self.called:
            # 当前timer被调用后
            # 设置被调用标记
            self.called = True
            cb, args, kw = self.tpl
            try:
                # 当cb是GreenThread/greenlet 的switch时
                # cb(*args, **kw)相当于
                # 执行的switch(*args, **kw)

                # 否则就是执行具体的外部函数或event
                cb(*args, **kw)
            finally:
                try:
                    # 删除self.tpl
                    # 防止self.tpl的存在导致绿色线程变量引用计数器不为0
                    # 最终导致绿色线程没被gc删除
                    del self.tpl
                except AttributeError:
                    pass

    def cancel(self):
        # 取消当前定时器,如果这个定时器已经被完成或者取消
        # 这个函数无效
        # 可以看出timer的cancel
        # 最终调用的是hub类的timer_canceled
        if not self.called:
            self.called = True
            get_hub().timer_canceled(self)
            try:
                del self.tpl
            except AttributeError:
                pass
    # 用于比较timer实例的大小
    # 方便timer组成的列表排序
    def __lt__(self, other):
        return id(self) < id(other)
```

我在来看一个关键类FdListener

```python
class FdListener(object):
    # 这个类是用于存放需要监听fd的绿色线程
    # 有greenlet和对应的fd
    # cb是外部传入的通知函数

    # cb如果是绿色线程的的switch,用于跳转回原来的绿色线程
    # 外部一般用trampoline生成FdListener实例
    # 在trampoline函数中,传入的cb是greenlet.getcurrent().switch
    # 也就说和self.greenlet.switch相同
    # cb也有可能是一个event函数,比价少见,zmq的封装里有用event
    # cb最好不要是一个外部的函数,参考timer的cb说明
    # tb就是绿色线程的throw,一般是出现异常后执行的函数

    def __init__(self, evtype, fileno, cb, tb, mark_as_closed):
        assert (evtype is READ or evtype is WRITE)
        self.evtype = evtype
        self.fileno = fileno
        self.cb = cb
        self.tb = tb
        self.mark_as_closed = mark_as_closed
        self.spent = False
        self.greenlet = greenlet.getcurrent()

    def __repr__(self):
        return "%s(%r, %r, %r, %r)" % (type(self).__name__, self.evtype, self.fileno,
                                       self.cb, self.tb)
    __str__ = __repr__

    def defang(self):
        # cb重定向到closed_callback
        # 这个defang的不在本篇中说明
        self.cb = closed_callback
        if self.mark_as_closed is not None:
            self.mark_as_closed()
        self.spent = True
```

现在可以来看Hub类了

```text
[GCC 4.4.7 20120313 (Red Hat 4.4.7-17)] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import eventlet
>>> from eventlet.support import greenlets as greenlet
>>> hub = eventlet.hubs.get_hub()
>>> current = greenlet.getcurrent()
>>> current.switch()
()
>>> print current
<greenlet.greenlet object at 0x7fc3fc805f50>
>>> print hub.greenlet
<greenlet.greenlet object at 0x7fc3f0d87f50>
>>> print hub.greenlet.parent
<greenlet.greenlet object at 0x7fc3fc805f50>
>>> print hub.greenlet.dead
False
>>> 

```


```python

# noop是一个默认的fd listener类(单例)
# 监听fd为0, 用来当默认值防止循环报错的,后面wait里用到
noop = FdListener(READ, 0, lambda x: None, lambda x: None, None)

class Hub(object):
    SYSTEM_EXCEPTIONS = (KeyboardInterrupt, SystemExit)
    READ = READ
    WRITE = WRITE
    def __init__(self, clock=time.time):
        # listeners很重要
        # evelnet通过monkey patch hack了原生的socket
        # 当被hack过的socket类调用connect、recv、send等函数的时候
        # 会通过函数trampoline调用Hub.add(具体看后面的add函数)
        # 将需要监听的fd加入到self.listeners中去
        # 顺便,eventlet没有重载socket.listen函数
        # 也就是socket.listen是原生的不被绿色线程管理
        # 具体参考eventlet.greenio.base.GreenSocket类
        # 他取代了socket.socket
        self.listeners = {READ: {}, WRITE: {}}
        self.secondaries = {READ: {}, WRITE: {}}
        self.closed = []
        # 默认时间计算函数用time.time
        self.clock = clock
        # 启动一个绿色线程,跑self.run的代码
        # 前面我们说了 hub是线程中唯一的(线程中的单例)
        # 所以这个greenlet就是main greenlet,用于跑main loop,也就是self.run
        # 所有greenthread.spawn*函数孵化的绿色线程的parent都是这个greenlet

        # main loop的绿色线程的parent也就是
        # get_hubs().greenlet.parent具体哪里来的看c源码才知道
        # 只知道这个绿色线程不能在一开始就调用get_hubs().switch
        # 因为main loop的绿色线程一开始是dead状态的
        self.greenlet = greenlet.greenlet(self.run)
        self.stopping = False
        self.running = False
        # 所有列表保存需要处理的定时器Timer实例
        # 这个列表每次都会被排序,下次运行时间最小的排第一位
        # 为了能最快的排序,代码里用了堆排序
        self.timers = []
        # next_timers是预备定时器
        # main loop里会先把self.next_timers里的定时器装入self.timers
        # 然后再调用fire_timers来处理self.timers列表中的定时器
        # next_timers的是给外部绿色线程取消自己定时器用的
        # 这样main loop就不用排序完后,调用定时器的时候才发现已经取消
        # 减少self.timers中的元素以便更快排序
        self.next_timers = []
        # lclass是FdListener类
        self.lclass = FdListener
        # 通过这个计数器激活清理timers和next_timers列表
        self.timers_canceled = 0
        # 调试参数不看
        self.debug_exceptions = True
        self.debug_blocking = False
        self.debug_blocking_resolution = 1
        # 上面是BaseHub的代码 epoll的hub多了下面一些属性
        self.poll = epoll()
        self.modify = self.poll.modify

    def block_detect_pre(self):
        # 调试用,启动调试后在fire_timers前捕获信号
        # fire_timers就是调用最小时间的timer.__call__
        tmp = signal.signal(signal.SIGALRM, alarm_handler)
        if tmp != alarm_handler:
            self._old_signal_handler = tmp
        arm_alarm(self.debug_blocking_resolution)

    def block_detect_post(self):
        # 调试用,启动调试后在fire_timers后捕获信号
        if (hasattr(self, "_old_signal_handler") and
                self._old_signal_handler):
            signal.signal(signal.SIGALRM, self._old_signal_handler)
        signal.alarm(0)

    # epoll.hub的wait函数
    def wait(self, seconds=None):
        # wait是self.run,也就是main loop里用于调用切换到其他绿色线程的
        readers = self.listeners[READ]
        writers = self.listeners[WRITE]

        # 没有监听任何fd
        if not readers and not writers:
            if seconds:
                sleep(seconds)
            return
        try:
            # 从epoll中获取事件
            presult = self.do_poll(seconds)
        except (IOError, select.error) as e:
            if get_errno(e) == errno.EINTR:
                return
            raise
        SYSTEM_EXCEPTIONS = self.SYSTEM_EXCEPTIONS

        if self.debug_blocking:
            self.block_detect_pre()
        # 用epoll的返回事件生成回调列表
        callbacks = set()
        for fileno, event in presult:
            if event & READ_MASK:
                # noop不用管,防止错误的默认对象
                # get返回FdListener类实例
                callbacks.add((readers.get(fileno, noop), fileno))
            if event & WRITE_MASK:
                callbacks.add((writers.get(fileno, noop), fileno))
            if event & select.POLLNVAL:
                self.remove_descriptor(fileno)
                continue
            if event & EXC_MASK:
                callbacks.add((readers.get(fileno, noop), fileno))
                callbacks.add((writers.get(fileno, noop), fileno))

        for listener, fileno in callbacks:
            try:
                # 跳回listener所在绿色线程
                # listener所在的这个绿色线程
                # 调用accept、connect、recv、send等函数后会通过trampoline
                # switch到main loop
                # 现在main loop在这里又回到之前绿色线程执行的位置
                # 也就是trampoline调用hub.switch()后的部分
                # 接下来会执行
                # hub.remove(listener)  
                # timer.cancel
                # 返回之前hub.switch()的返回值
                # 也就是说最终又回到accept、connect、recv、send等调用的位置
                # 也就是在这里实现了不修改源代码也能绿化
                # 我们后面在详细看下trampoline函数
                listener.cb(fileno)
            except SYSTEM_EXCEPTIONS:
                raise
            except:
                self.squelch_exception(fileno, sys.exc_info())
                clear_sys_exc_info()
        #  其他绿色线程执行完回到这里
        if self.debug_blocking:
            self.block_detect_post()

    # epoll多出的函数    
    def do_poll(self, seconds):
        # epoll poll事件
        return self.poll.poll(seconds)

    def register(self, fileno, new=False):
        # 省略epoll注册监听的代码
        .....

    def add(self, evtype, fileno, cb, tb, mark_as_closed):
        oldlisteners = bool(self.listeners[READ].get(fileno) or
                            self.listeners[WRITE].get(fileno))
        # 这个代码用每次有socket调用accept、connect、recv、send等函数时
        # 最终都会触发这个Hub.add函数
        # 这个函数会通过fd和绿色线程生成FdListener实例
        # 并添加到self.listeners字典中
        # mark_as_closed是一个外部传入的函数
        # 用于fd关闭后的清理
        listener = self.lclass(evtype, fileno, cb, tb, mark_as_closed)
        bucket = self.listeners[evtype]
        if fileno in bucket:
            if g_prevent_multiple_readers:
                # raise内容我删除了
                raise RuntimeError(".....")
            self.secondaries[evtype].setdefault(fileno, []).append(listener)
        else:
            bucket[fileno] = listener
        # 下面是epoll多出来的,epoll注册监听
        # 可以看到每次调用Hub.add都会调用epoll的注册fd
        try:
            if not oldlisteners:
                self.register(fileno, new=True)
            else:
                self.register(fileno, new=False)
        except IOError as ex:    # ignore EEXIST, #80
            # 忽略fd已经注册过的错误
            if get_errno(ex) != errno.EEXIST:
                raise
        return listener

    def remove(self, listener):
        # 这个函数也是用于外部调用
        # 用于移除监听的fd
        if listener.spent:
            # trampoline may trigger this in its finally section.
            return
        fileno = listener.fileno
        evtype = listener.evtype
        self.listeners[evtype].pop(fileno, None)
        # migrate a secondary listener to be the primary listener
        if fileno in self.secondaries[evtype]:
            sec = self.secondaries[evtype].get(fileno, None)
            if not sec:
                return
            self.listeners[evtype][fileno] = sec.pop(0)
            if not sec:
                del self.secondaries[evtype][fileno]

    def remove_descriptor(self, fileno):
        # 用注销fd监听,
        # hub中从self.listeners弹出fd
        # 再切换到fd监听的绿色线程做对应清理工作
        # 由内部catch的函数调用
        listeners = []
        listeners.append(self.listeners[READ].pop(fileno, noop))
        listeners.append(self.listeners[WRITE].pop(fileno, noop))
        listeners.extend(self.secondaries[READ].pop(fileno, ()))
        listeners.extend(self.secondaries[WRITE].pop(fileno, ()))
        for listener in listeners:
            try:
                # 看wait函数里相同部分的说明
                listener.cb(fileno)
            except Exception:
                self.squelch_generic_exception(sys.exc_info())

    def close_one(self):
        # 处理已经关闭的端口的绿色线程
        listener = self.closed.pop()
        if not listener.greenlet.dead:
            # There's no point signalling a greenlet that's already dead.
            listener.tb(IOClosed(errno.ENOTCONN, "Operation on closed file"))

    def ensure_greenlet(self):
        # hub的绿色线程挂了
        # 也就是main loop所在线程挂了
        # 刚开始Hub的main loop是dead的
        if self.greenlet.dead:
            # 重新开一个绿色线程
            new = greenlet.greenlet(self.run, self.greenlet.parent)
            # 设置self.greenlet.parent和self.greenlet为新的绿色线程
            self.greenlet.parent = new
            self.greenlet = new

    def switch(self):
        # 这个switch用于给其他绿色线程放弃cpu时间
        # 返回到main loop(也就是这里的self.run里)的绿色线程中的函数
        # 用法是其他绿色线程调用get_hub().switch()
        # 当然其他绿色线程也可以调用self.parent.switch()来回到main loop
        # hup里写的switch封装了一些其他东西,比如main loop重启
        # 所以切换到main loop用get_hub().switch()比较规范
        cur = greenlet.getcurrent()
        # main loop所在绿色线程不能执行这个switch函数
        assert cur is not self.greenlet, 'Cannot switch to MAINLOOP from MAINLOOP'
        switch_out = getattr(cur, 'switch_out', None)
        if switch_out is not None:
            try:
                switch_out()
            except:
                self.squelch_generic_exception(sys.exc_info())
        # 确保main loop没有挂
        # 如果main loop挂了就重新启动main loop及其对应绿色线程
        self.ensure_greenlet()
        try:
            # 每次都把非main loop的绿色线程parent设置为main loop的绿色线程
            # 这里要配合self.ensure_greenlet才好理解
            # 在main loop重启后,所有绿色线程的parent都要重新指向
            # 新main loop所在的绿色线程
            if self.greenlet.parent is not cur:
                cur.parent = self.greenlet
        except ValueError:
            pass  # gets raised if there is a greenlet parent cycle
        clear_sys_exc_info()
        return self.greenlet.switch()

    def default_sleep(self):
        return 60.0

    def sleep_until(self):
        t = self.timers
        if not t:
            return None
        return t[0][0]

    def run(self, *a, **kw):
        # 这个run就是hub的循环
        # 前面我们说了 hub是线程中唯一的(线程中的单例)
        # 这个循环就是main loop
        # 当其他线程调用self.parent.switch()或者调用get_hub().switch()
        # 的时候就会切换到当前循环中
        if self.running:
            raise RuntimeError("Already running!")
        try:
            # 循环中删除部分调试代码
            self.running = True
            self.stopping = False
            while not self.stopping:
                # 清理所有已经关闭的fd
                while self.closed:
                    # 一次清理一个
                    self.close_one()
                # 看函数说明
                self.prepare_timers()
                # 看函数说明
                self.fire_timers(self.clock())
                self.prepare_timers()
                # wakeup_when是通过self.timers[0]的执行时间来计算sleep_time
                wakeup_when = self.sleep_until()
                if wakeup_when is None:
                    sleep_time = self.default_sleep()
                else:
                    sleep_time = wakeup_when - self.clock()
                # wait里切换到处理epoll读写事件的fd的绿色线程
                # 也就是说main loop里有两次会切换其他绿色线程中
                # 一次是fire_timers、一次是wait
                if sleep_time > 0:
                    self.wait(sleep_time)
                else:
                    self.wait(0)
            # 下面是循环结束后的清理代码
            else:
                self.timers_canceled = 0
                del self.timers[:]
                del self.next_timers[:]
        finally:
            self.running = False
            self.stopping = False

    def add_timer(self, timer):
        # 把一个定时器加入到next_timers列表中
        # next_timers列表是用来预备调度的定时器列表
        # main loop中会通过prepare_timers函数讲
        # next_timers推送到timers列表中
        # timers列表中的timer会被fire_timers函数执行
        scheduled_time = self.clock() + timer.seconds
        self.next_timers.append((scheduled_time, timer))
        return scheduled_time

    def timer_canceled(self, timer):
        # timer取消,已经完成的timer也是使用timer_canceled
        self.timers_canceled += 1
        len_timers = len(self.timers) + len(self.next_timers)
        # 要执行的timers和准备要执行的timer加起来超过1000
        # 且timers_canceled计数器大于上述长度一半的时候
        # 开始清理掉有called标记的timer
        # 可以看出timers_canceled是一个清理用的计数器
        if len_timers > 1000 and len_timers / 2 <= self.timers_canceled:
            self.timers_canceled = 0
            self.timers = [t for t in self.timers if not t[1].called]
            self.next_timers = [t for t in self.next_timers if not t[1].called]
            # 重新排序
            heapq.heapify(self.timers)

    def prepare_timers(self):
        # 把next_timers中的所有定时器放入timers列表中
        # 推入的时候已经完成排序
        # 时间最短的timer在0号位置
        heappush = heapq.heappush
        t = self.timers
        for item in self.next_timers:
            if item[1].called:
                self.timers_canceled -= 1
            else:
                heappush(t, item)
        # 清空next_timers列表
        del self.next_timers[:]

    def fire_timers(self, when):
        # 从self.timers取出定时器然后执行
        t = self.timers
        heappop = heapq.heappop
        # 在timers列表不为空的情况下循环
        while t:
            # next是timers中第一个元素
            # next是一个tuple,0 是timer实例,1是这个timer需要执行的时间点
            # timers是排序过的,第一个元素肯定是时间属性最小的
            next = t[0]
            # exp是timer需要执行的时间点
            exp = next[0]
            timer = next[1]
            # 第一个需要执行的定时器都还没到需要执行的时间
            # 直接退出循环
            if when < exp:
                break
            # 这里相当于t.pop(0)
            # 然后再排序把最小的放最前面
            # 下次取t[0]的时候还是最小
            # 反正是堆排序
            heappop(t)
            try:
                # 取出来的timer有called标记
                if timer.called:
                    # timers_canceled计数器减1
                    self.timers_canceled -= 1
                else:
                    # 这里是Time.__call__
                    # 一般是switch到对应绿色线程
                    timer()
            except self.SYSTEM_EXCEPTIONS:
                raise
            except:
                self.squelch_timer_exception(timer, sys.exc_info())
                clear_sys_exc_info()

    def schedule_call_global(self, seconds, cb, *args, **kw):
        # 把一个绿色线程/函数封装为timer, cb就是这个函数或者绿色线程的switch
        # 并加入到next_timers预备定时器列表中
        # timer被main loop调用的时候
        # 会执行cb(*args, **kw)
        t = timer.Timer(seconds, cb, *args, **kw)
        self.add_timer(t)
        return t
    # 省略部分调试代码
    ......
```

---

关键的trampoline函数,monkey patch过的socket最终通过trampoline函数来调用Hub.add的

```python
def trampoline(fd, read=None, write=None, timeout=None,
               timeout_exc=timeout.Timeout,
               mark_as_closed=None):
    t = None
    hub = get_hub()
    current = greenlet.getcurrent()
    assert hub.greenlet is not current, 'do not call blocking functions from the mainloop'
    # 不能同时又read又wirte
    assert not (
        read and write), 'not allowed to trampoline for reading and writing'
    # 获取到socket的fileno
    try:
        fileno = fd.fileno()
    except AttributeError:
        fileno = fd
    # 当外部没有传入timeout的时候才会添加这个调试用的定时器
    if timeout is not None:
        def _timeout(exc):
            # This is only useful to insert debugging
            # 注释表示这里是为了插入调试
            current.throw(exc)
        # 这里可以看出schedule_call_global的cb不一定是switch
        # 这个定时器用于触发current.throw
        t = hub.schedule_call_global(timeout, _timeout, timeout_exc)
    try:
        # 这里调用hub.add
        # 这里可以看到,每次socket调用后才调用hub.add
        # 调用完又把fd从hub的listeners里删除
        # hub里的epoll会不停的注册、注销fd
        if read:
            listener = hub.add(hub.READ, fileno, current.switch, current.throw, mark_as_closed)
        elif write:
            listener = hub.add(hub.WRITE, fileno, current.switch, current.throw, mark_as_closed)
        try:
            # 这里的finally有点晕人
            # 写成  
            #  ret = hub.switch()
            #  return ret
            # 这样顺序就好理解一点了
            # 先 switch 切到main loop
            # 当被切回来的时候
            # remove、然后再 t.cancel
            # 然后return ret
            # ret值当然是None了
            # 这里的返回值也没什么用
            return hub.switch()
        finally:
            hub.remove(listener)
    finally:
        if t is not None:
            t.cancel()
```

---

我们以recv为例来看看GreenSocket怎么封装的

```python
class GreenSocket(object):
    # 这个类用于hack  socket.socket实例
    # 在外部调用socket.socket的时候实际上在调用GreenSocket
    def __init__(self, family_or_realsock=socket.AF_INET, *args, **kwargs):

        if isinstance(family_or_realsock, six.integer_types):
            fd = _original_socket(family_or_realsock, *args, **kwargs)
            # Notify the hub that this is a newly-opened socket.
            notify_opened(fd.fileno())
        else:
            fd = family_or_realsock
        # 这里的fd,就是真正的socket.socket实例
        self.fd = fd
        ...

    def recv(self, bufsize, flags=0):
        return self._recv_loop(self.fd.recv, bufsize, flags)

    def _recv_loop(self, recv_meth, *args):
        fd = self.fd
        # 使用非阻塞socket的时候
        # 调用self.fd.recv
        # 也就是直接调用socket.recv
        # 没有绿色线程什么事
        if self.act_non_blocking:
            return recv_meth(*args)
        # 非阻塞的recv走这里
        while True:
            try:
                # recv: bufsize=0?
                # recv_into: buffer is empty?
                # This is needed because behind the scenes we use sockets in
                # nonblocking mode and builtin recv* methods. Attempting to read
                # 0 bytes from a nonblocking socket using a builtin recv* method
                # does not raise a timeout exception. Since we're simulating
                # a blocking socket here we need to produce a timeout exception
                # if needed, hence the call to trampoline.
                # 因为说明就不翻译了
                # 这里的判断可以看出!!!!设置了recv的长度,会不走绿色线程!!!
                if not args[0]:
                    # 这里是封装了的trampoline函数
                    # 这里走完表示epoll在当前fd有事件
                    # 从main loop回到当前绿色线程
                    self._read_trampoline()
                # 因为是有epoll事件的,所以这里自然不会阻塞
                # 但这里的recv时间太长会影响整个循环
                # 不过socket.recv的时间应该不会太长
                return recv_meth(*args)
            except socket.error as e:
                # 这个错误是connect还没建立完成就recv的错误
                if get_errno(e) in SOCKET_BLOCKING:
                    pass
                elif get_errno(e) in SOCKET_CLOSED:
                    return b''
                else:
                    raise
            # 遇到SOCKET_BLOCKING错误走这里
            try:
                self._read_trampoline()
            except IOClosed as e:
                # Perhaps we should return '' instead?
                raise EOFError()

    def _read_trampoline(self):
        self._trampoline(
            self.fd,
            read=True,
            timeout=self.gettimeout(),
            timeout_exc=socket.timeout("timed out"))

    def _trampoline(self, fd, read=False, write=False, timeout=None, timeout_exc=None):
        if self._closed:
            # If we did any logging, alerting to a second trampoline attempt on a closed
            # socket here would be useful.
            raise IOClosed()
        try:
            # 这里调用trampoline,走绿色线程
            return trampoline(fd, read=read, write=write, timeout=timeout,
                              timeout_exc=timeout_exc,
                              mark_as_closed=self._mark_as_closed)
        except IOClosed:
            # This socket's been obsoleted. De-fang it.
            self._mark_as_closed()
            raise

```

我们现在来总结下,我们先简化一下Hub的man loop

```python
while not self.stopping:
    self.prepare_timers()
    self.fire_timers(self.clock())
    self.prepare_timers()
    self.wait(0)
```


Hup的main loop里主要的工作1:fire_timers,处理定时器

    定时器到点就调用timer中的cb
    定时器的添加一般通过Hub.schedule_call_global
    一般来说你的函数执行过程中要等待另外一个函数执行完
    就通过eventlet.greenthread.sleep添加定时器并切换到其他绿色线程

我们来看看sleep的实现

```python
def sleep(seconds=0):
    hub = hubs.get_hub()
    current = getcurrent()
    assert hub.greenlet is not current, 'do not call blocking functions from the mainloop'
    timer = hub.schedule_call_global(seconds, current.switch)
    try:
        hub.switch()
    finally:
        # 切换回来的时候删除添加的timer
        timer.cancel()
```

Hup的main loop里主要的工作2:wait,通过epoll处理fd事件

    处理完定时器后,检查self.listeners字典
    如果里面有内容,就调用epoll扫描字典中的fd
    如果fd中有事件，切换到fd相关的绿色线程
    listeners的添加一般通过Hub.add或者封装过的trampoline函数

    monkey patch后的原生socket.socket等被替换
    执行诸如socket.recv之类的函数最终会根据情况调用trampoline
    所以openstack可以在不修改pika、wsgify的socket数据处理部分代码的情况下绿化它们

到这里,我们就大致的理解了eventlet的Hub工作原理了,eventlet也就明白了一半,再深入就要看[eventlet中event的工作原理](http://www.lolizeppelin.com/2017/03/14/python-eventlet-event/)
