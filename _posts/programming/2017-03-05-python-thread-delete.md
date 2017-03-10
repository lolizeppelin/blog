---
layout: post
title:  "python threading.Thread删除"
date:   2017-03-05 12:50:00 +0800
categories: "编程"
tag: ["python", "linux"]
---

* content
{:toc}


程序用了老早的一个ThreadPool模块,站点用的uwsgi方式运行,运行时间久了以后发现有时候会出现无法分配线程

问题肯定在于线程join后依然没有释放,研究了下thread的线程的销毁

首先解决一个语法问题,我来看

```python

_shutdown = _MainThread()._exitfunc

class _MainThread(Thread):

    def _exitfunc(self):
        ....    
        # Thread中没有_Thread__delete这个方法
        self._Thread__delete()
```


原因在于

```python
class A(object):
    def _internal_use(self):
        pass
    def __method_name(self):
        pass
dir(A())
['_A__method_name', ..., '_internal_use']
# 也就是说

# _Thread__delete() 就是 __delete()

```
一个不错的关于下划线的文章[参考](http://python.jobbole.com/81129/)


然后,代码中没有找到调用delete方法销毁线程的地方,这也是我们最关心的

找到一篇相关[文章](http://blog.csdn.net/yueguanghaidao/article/details/40088431)

大致就是

    谁调用shutdown函数的呢？
    当python要销毁运行时之前肯定会调用
    所以打开pythonrun.c，你会发现如下函数


```python
/* Wait until threading._shutdown completes, provided
   the threading module was imported in the first place.
   The shutdown routine will wait until all non-daemon
   "threading" threads have completed. */  
static void  
wait_for_thread_shutdown(void)  
{  
#ifdef WITH_THREAD  
    PyObject *result;  
    PyThreadState *tstate = PyThreadState_GET();  
    PyObject *threading = PyMapping_GetItemString(tstate->interp->modules,  
                                                  "threading");  
    if (threading == NULL) {  
        /* threading not imported */  
        PyErr_Clear();  
        return;  
    }  
    result = PyObject_CallMethod(threading, "_shutdown", "");  
    if (result == NULL)  
        PyErr_WriteUnraisable(threading);  
    else  
        Py_DECREF(result);  
    Py_DECREF(threading);  
#endif  
}  
```

既然python的虚拟机是这样调用的,所以我们手动销毁线程也就跟着写就好了

下面是改动过的ThreadPool的线程join方法

```python

def dismissWorkers(self, num_workers, do_join=False):
    """Tell num_workers worker threads to quit after their current task."""
    dismiss_list = []
    for i in range(min(num_workers, len(self.workers))):
        worker = self.workers.pop()
        worker.dismiss()
        dismiss_list.append(worker)

    if do_join:
        while dismiss_list:
            worker = dismiss_list.pop()
            worker.join()
            # 清理线程
            worker._Thread__delete()
            del worker
    else:
        self.dismissedWorkers.extend(dismiss_list)


def joinAllDismissedWorkers(self):
    """Perform Thread.join() on all worker threads that have been dismissed.
    """
    while self.dismissedWorkers:
        worker = self.dismissedWorkers.pop()
        worker.join()
        # 清理线程
        worker._Thread__delete()
        del worker
```
