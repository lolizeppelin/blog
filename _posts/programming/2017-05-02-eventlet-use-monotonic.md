---
layout: post
title:  "python eventlet使用monotonic"
date:   2017-05-02 12:50:00 +0800
categories: "编程"
tag: ["python", "linux"]
---

* content
{:toc}


写了这么一个优先级锁用来取代openstack里rabbit驱动的connection里用的ConnectionLock

```python
class DummyLock(object):
    def acquire(self):
        pass

    def release(self):
        pass

    def __enter__(self):
        self.acquire()

    def __exit__(self, type, value, traceback):
        self.release()

    def set_defalut_priority(self, priority):
        pass

    @contextlib.contextmanager
    def priority(self, priority):
        try:
            yield
        finally:
            self.release()


class PriorityGreenlet(object):

    def __init__(self, priority, greenlet):
        self.priority = priority
        self.greenlet = greenlet

    def __lt__(self, other):
        # if self.priority == priority.level:
        #     return id(self.greenlet) < id(other.greenlet)
        return self.priority < other.priority


class PriorityLock(DummyLock):
    """lock with priority
    a copy of Semaphore
    """
    def __init__(self):
        self.locked = False
        self._waiters = []
        self.default_priority = 0
        self.priority_lock = {}

    def set_defalut_priority(self, priority):
        if not isinstance(priority, int) or self.locked:
            raise RuntimeError('Priority not int or Lock is Locked')
        self.default_priority = priority

    def release(self):
        self.locked = False
        if self._waiters:
            waiter = heapq.heappop(self._waiters)
            hub.schedule_call_global(0, waiter.greenlet.switch)

    def acquire(self):
        self.acquire_with_priority(self.default_priority)

    def acquire_with_priority(self, priority):
        current_thread = eventlet.getcurrent()
        for _waiters in self._waiters:
            # 避免当前线程再锁
            if current_thread is _waiters.greenlet:
                return
        if self.locked:
            locker = PriorityGreenlet(priority, current_thread)
            heapq.heappush(self._waiters, locker)
            print 'i will go out'
            hub.switch()
            print 'i am switched', priority
            # self._waiters.remove(locker)
        else:
            print 'not lock'
        self.locked = True

    @contextlib.contextmanager
    def priority(self, priority):
        self.acquire_with_priority(priority)
        try:
            yield
        finally:
            self.release()
```

之后用如下方法测试的时候出现了问题
```python
import eventlet
import eventlet.hubs
from simpleutil.utils.lockutils import PriorityLock


lock = PriorityLock()

lock.set_defalut_priority(2)
lock.set_defalut_priority(0)

# with lock:
#     print 'get lock success'
#
# with lock.priority(1):
#     print 'get new_lock success\n\n'

def locker_0():
    print 'start lock 0'
    with lock:
        print 'locker 0 success'
        eventlet.sleep(1)
        print 'locker 0 unlock'

def locker_1():
    with lock.priority(1):
        print 'get lokcer_1'

def locker_2():
    with lock.priority(2):
        print 'get lokcer_2'

def locker_3():
    with lock.priority(3):
        print 'get lokcer_3'


eventlet.spawn_n(locker_0)
eventlet.spawn_n(locker_3)
eventlet.spawn_n(locker_2)
eventlet.spawn_n(locker_1)
eventlet.spawn_n(locker_2)
eventlet.spawn_n(locker_3)
eventlet.sleep(3)
```

偶尔出现输出为
```text
not lock
get lokcer_3
not lock
get lokcer_2
not lock
get lokcer_1
start lock 0
not lock
locker 0 success
i will go out
i will go out
locker 0 unlock
i am switched 2
get lokcer_2
i am switched 3
get lokcer_3
```

上述输出表明, 第一个锁在lock完成前,第二、三、四3个个锁居然执行了,上述问题情况在单线程情况下不应该出现的......

想了一下终于想明白了,在spawn_n(locker_0)后加上eventlet.sleep(0),问题果然就不出现了,原因在于

    hub内部是通过每个timer的时间点来排序的,这个时间点是通过time.time生成的
    上述代码在“同一”时间spawn_n,如果time.time的精度不够,那么后spawn_n生成的timer的
    时间就会和前面生成的timer的时间点相同,造成排序的时候有可能后生成的timer排到了前面

    所以,上述解决方案并不是一个根本方法,只保证了第一个锁的时间点,所以要根本解决上述问题
    方法一： 每个spawn_n后跟一个sleep
    方法二： 使用spawn而不是spawn_n,因为spawn封装的内容多一些,最终添加timer的时间点
            自然也就不会相同了
    方法一和方法二在cpu很快的情况下依旧可能出问题

    方法三： 用高精时间取代time.time, 例如monotonic(默认从librt中获取)

重写下基本Hub

```python
import eventlet.hubs.selects
from simpleutil.utils.timeutils import monotonic

class BaseHub(eventlet.hubs.selects.Hub):

    def __init__(self):
        super(BaseHub, self).__init__(clock=monotonic)

eventlet.hubs.use_hub(BaseHub)
eventlet.hubs.get_hub()
```

经过测试...果然解决了排序错误导致执行顺序错误的问题..看上去问题解决了以后,顺手测试了下效率,结果

    所有测试都是分别执行30000000次monotonic()和time.time(),结果如下
    windows 10   python 2.7
    15.6640000343
    4.8180000782

    centos7 python 2.7
    150.870907068
    14.4426150322

    centos 6 python 2.6
    92.3898830414
    6.76288104057

所以python调用c还是存在效率问题的

回头想想这个bug只影响了spawn_n的时候锁获取的顺序,但是锁还是正常工作的,所以不需要在意顺序的情况下,这个bug代码工作没有影响

所以....还是什么都不要改比较好
