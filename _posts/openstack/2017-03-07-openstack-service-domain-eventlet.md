---
layout: post
title:  "OpenStack Mitaka从零开始 openstack的守护进程实现过程"
date:   2016-12-06 15:05:00 +0800
categories: "虚拟化"
tag: ["openstack", "python", "linux"]
---

* content
{:toc}



1. launcher  
> openstack里的launcher有两种(在oslo_service.service中),
> ServiceLauncher和ProcessLauncher,一般用法为

```python
def launch(conf, service, workers=1):
    if workers is None or workers == 1:
        launcher = ServiceLauncher(conf)
        launcher.launch_service(service)
    else:
        launcher = ProcessLauncher(conf)
        launcher.launch_service(service, workers=workers)
    return launcher
```

由于各个组件是不同人写的,写法有点不同,上面写法是neutorn里的比较标准

nova里的为了单例绕了一下就难看一点

```python
# 用这个来做单例
_launcher = None
# 外部代码里调用的是serve而不是launch比较难看,neutron里的比较好看
def serve(server, workers=None):
    global _launcher
    if _launcher:
        raise RuntimeError(_('serve() can only be called once'))
    # 就是前面的launch函数
    _launcher = service.launch(CONF, server, workers=workers)

def wait():
    _launcher.wait()
```


一般都使用多进程的ProcessLauncher,我们来看下核心代码

```python

class ServiceWrapper(object):
    def __init__(self, service, workers):
        # service对象
        self.service = service
        # 总共可fork的进程数量
        self.workers = workers
        # 存放子进程pid
        self.children = set()
        # 存放每次fork出子进程的用时
        # 用于防止fork进程过快
        self.forktimes = []


class ProcessLauncher(object):
    """Launch a service with a given number of workers."""

    def __init__(self, conf, wait_interval=0.01):
        ...
        # 子进程字典,pid为key
        self.children = {}
        # 运行标志,实例化的时候就认为已经启动
        self.running = True
        # 循环间隔
        self.wait_interval = wait_interval
        # 这launcher是子进程里用的,主进程中无用
        self.launcher = None
        # eventlet通信管道
        rfd, self.writepipe = os.pipe()
        self.readpipe = eventlet.greenio.GreenPipe(rfd, 'r')
        # 信号处理
        self.signal_handler = SignalHandler()
        self.handle_signal()

    def launch_service(self, service, workers=1):
        # 这个函数是一个初始化函数只运行一次
        # 这里检查service的类型是不是继承于ServiceBase
        _check_service_base(service)
        # 生成wrap, 看前面ServiceWrapper类,里面有部分属性没有方法
        wrap = ServiceWrapper(service, workers)
        LOG.info(_LI('Starting %d workers'), wrap.workers)
        # 生成工作的worker,也就是这里fork出多个进程
        # 这里可以看出ServiceWrapper里面保存了所有的子进程的pid之类的信息
        while self.running and len(wrap.children) < wrap.workers:
            self._start_child(wrap)

    def _start_child(self, wrap):
        ...
        # 前面跳过防止fork过快的代码
        pid = os.fork()
        if pid == 0:
            # 子进程中给launcher赋值
            self.launcher = self._child_process(wrap.service)
            # 自动重启launcher的循环
            while True:
                # 子进程注册进程接受信号
                self._child_process_handle_signal()
                # 接收到信号返回,未收到信号的时候进程阻塞于此,内有死销魂还
                status, signo = self._child_wait_for_exit_or_signal(
                    self.launcher)
                # 收到正常退出信号,子进程结束
                if not _is_sighup_and_daemon(signo):
                    self.launcher.wait()
                    break
                # 重启sefl.launcher
                self.launcher.restart()
            os._exit(status)
        # 主进程
        LOG.debug('Started child %d', pid)
        wrap.children.add(pid)
        self.children[pid] = wrap
        return pid

    # 子进程才用到下面代码

    def _child_process(self, service):
        # 这里注册一次信号接收函数
        # 后面又注册一次不是很理解
        self._child_process_handle_signal()
        # 后面就是eventlet相关代码
        # Reopen the eventlet hub to make sure we don't share an epoll
        # fd with parent and/or siblings, which would be bad
        eventlet.hubs.use_hub()
        # Close write to ensure only parent has it open
        os.close(self.writepipe)
        # Create greenthread to watch for parent to close pipe
        eventlet.spawn_n(self._pipe_watcher)
        # Reseed random number generator
        random.seed()
        # 这里就是调用Launcher,注意不是ProcessLauncher
        launcher = Launcher(self.conf)
        launcher.launch_service(service)
        return launcher


    def _child_wait_for_exit_or_signal(self, launcher):
        status = 0
        signo = 0
        try:
            # 子进程的工作循环在这里
            # 这个launcher不是ProcessLauncher
            launcher.wait()
        except SignalExit as exc:
            signame = self.signal_handler.signals_to_name[exc.signo]
            LOG.info(_LI('Child caught %s, exiting'), signame)
            status = exc.code
            signo = exc.signo
        except SystemExit as exc:
            status = exc.code
        except BaseException:
            LOG.exception(_LE('Unhandled exception'))
            status = 2
        return status, signo

    # 上面是子进程用到的代码
    # 下面是主进程用到的代码

    def wait(self):
        # 这里是主进程的wait死循环部分,不是self.launcher的wait部分
        # 通知systemd
        systemd.notify_once()
        if self.conf.log_options:
            LOG.debug('Full set of CONF:')
            self.conf.log_opt_values(LOG, logging.DEBUG)
        try:
            # 这个循环不是工作循环
            # 这个是用于进程崩溃后自动重启的
            while True:
                self.handle_signal()
                # 死循环在此,这个函数用于重启子进程
                self._respawn_children()
                # 走到这里表示主进程收到退出信号
                if not self.sigcaught:
                    return
                # 省略部分是处理崩溃的
                ....
        except eventlet.greenlet.GreenletExit:
            LOG.info(_LI("Wait called after thread killed. Cleaning up."))
        # 后面是退出以及清理部分
        ....

    def _respawn_children(self):
        # 重启子进程的函数
        while self.running:
            # 里面主要是调用os.waitpid获取子进程的返回
            wrap = self._wait_child()
            # 因为子进程没有退出所有正常都是走到这里
            if not wrap:
                # 这个sleep比较特别
                # 后面在看这个sleep
                eventlet.greenthread.sleep(self.wait_interval)
                continue
            # 有子进程退出,重新fork出一个worker子进程
            while self.running and len(wrap.children) < wrap.workers:
                self._start_child(wrap)

```

下面我们来看Launcher和Services, Launcher也就是ProcessLauncher中的self.Launcher

```python
class Launcher(object):

    def __init__(self, conf):
        self.conf = conf
        conf.register_opts(_options.service_opts)
        # 创建一个services类
        # services类用于管理service
        self.services = Services()
        # 后门端口
        self.backdoor_port = (
            eventlet_backdoor.initialize_if_enabled(self.conf))

    def launch_service(self, service):
        # 这里和ProcessLauncher中一样先检查service的类型
        _check_service_base(service)
        service.backdoor_port = self.backdoor_port
        # 添加到services中,这里其实就是绿色线程spawn开始的地方
        self.services.add(service)

    def stop(self):
        self.services.stop()

    def wait(self):
        self.services.wait()

    def restart(self):
        self.conf.reload_config_files()
        self.services.restart()

# 由此可见,绿色线程关键启动在Services中

class Services(object):

    def __init__(self):
        # 一个存放service的列表
        self.services = []
        # 初始化一个threadgroup.ThreadGroup()
        self.tg = threadgroup.ThreadGroup()
        # 先别管这个event对象
        self.done = event.Event()

    def add(self, service):
        # 当我们调用add的时候
        # 就是调用了ThreadGroup的add_thread方法
        # self.run_service是函数, service, self.done是
        # self.run_service的参数
        self.services.append(service)
        self.tg.add_thread(self.run_service, service, self.done)

    def stop(self):
        """Wait for graceful shutdown of services and kill the threads."""
        for service in self.services:
            service.stop()

        # Each service has performed cleanup, now signal that the run_service
        # wrapper threads can now die:
        if not self.done.ready():
            self.done.send()

        # reap threads:
        self.tg.stop()

    def wait(self):
        """Wait for services to shut down."""
        for service in self.services:
            service.wait()
        self.tg.wait()

    def restart(self):
        """Reset services and start them in new threads."""
        self.stop()
        self.done = event.Event()
        for restart_service in self.services:
            restart_service.reset()
            self.tg.add_thread(self.run_service, restart_service, self.done)

    @staticmethod
    def run_service(service, done):
        # 这里调用了service的start方法并wait
        try:
            service.start()
        except Exception:
            LOG.exception(_LE('Error starting thread.'))
            raise SystemExit(1)
        else:
            done.wait()


# 我们来看ThreadGroup是什么

class ThreadGroup(object):
    def __init__(self, thread_pool_size=10):
        # eventlet 绿色线程的池在这里初始化
        self.pool = greenpool.GreenPool(thread_pool_size)
        self.threads = []
        self.timers = []

    def add_thread(self, callback, *args, **kwargs):
        # 也就是说前面的add方法最终调用了
        # 绿色线程池的spawn方法
        gt = self.pool.spawn(callback, *args, **kwargs)
        th = Thread(gt, self)
        self.threads.append(th)
        return th

```

接下来就是复杂的eventlet的分析了
