---
layout: post
title:  "python eventlet中event的工作原理"
date:   2017-03-14 12:50:00 +0800
categories: "编程"
tag: ["python", "linux"]
---

* content
{:toc}


我们来看event类

```python

class NOT_USED:
    def __repr__(self):
        return 'NOT_USED'

# 这个单例子用来做is判断
# 比直接用字符串比较来的快
# 还能输出字符串
# 又学习到新方法
NOT_USED = NOT_USED()

class Event(object):
    _result = None
    _exc = None

    def __init__(self):
        # 一个列表用于存放正在等待的绿色线程
        # 这里可以看出这个Event是可能被多个绿色线程调用
        self._waiters = set()
        # 初始化self._result、self._exc值
        self.reset()

    def __str__(self):
        params = (self.__class__.__name__, hex(id(self)),
                  self._result, self._exc, len(self._waiters))
        return '<%s at %s result=%r _exc=%r _waiters[%d]>' % params

    def reset(self):
        # 必须在  self._result不是NOT_USED的时候调用
        assert self._result is not NOT_USED, 'Trying to re-reset() a fresh event.'
        self._result = NOT_USED
        self._exc = None

    def ready(self):
        # 当 self._result 不是 NOT_USED的时候
        # init后self._result == NOT_USED
        return self._result is not NOT_USED

    def has_exception(self):
        return self._exc is not None

    def has_result(self):
        return self._result is not NOT_USED and self._exc is None

    def poll(self, notready=None):
        if self.ready():
            return self.wait()
        return notready

    def poll_exception(self, notready=None):
        # 用于poll错误
        if self.has_exception():
            return self.wait()
        return notready

    def poll_result(self, notready=None):
        # 用于推送result
        if self.has_result():
            return self.wait()
        return notready

    def wait(self):
        # 绿色线程GreenThread调用wait的时候会走到这里
        current = greenlet.getcurrent()
        # _result还是默认标记
        if self._result is NOT_USED:
            # 把当前绿色线程放入_waiters这个list中
            self._waiters.add(current)
            try:
                # 先切换到main loop
                # 切换回来的时候
                # 把_waiters中的当前绿色线删除(discard方法不报错)
                return hubs.get_hub().switch()
            finally:
                self._waiters.discard(current)
        # 如果self._exc不为空通过绿色线程throw异常
        if self._exc is not None:
            current.throw(*self._exc)
        return self._result

    def send_exception(self, *args):
        # 发送一个exception
        return self.send(None, args)

    def send(self, result=None, exc=None):
        #  self._result还是默认标记才能send
        assert self._result is NOT_USED, 'Trying to re-send() an already-triggered event.'
        # 设置_result
        self._result = result
        if exc is not None and not isinstance(exc, tuple):
            exc = (exc, )
        self._exc = exc
        hub = hubs.get_hub()
        # _waiters中所有绿色线程作为参数
        # 创建多个定时器
        for waiter in self._waiters:
            hub.schedule_call_global(
                0, self._do_send, self._result, self._exc, waiter)

    def _do_send(self, result, exc, waiter):
        # 在hub中fire_timers的cb调用
        # 这里再次判断waiter是否还在_waiters中
        # 因为可能会被其他绿色线程cancel掉
        if waiter in self._waiters:
            if exc is None:
                waiter.switch(result)
            else:
                waiter.throw(*exc)
```
