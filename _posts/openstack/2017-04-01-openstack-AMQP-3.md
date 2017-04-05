---
layout: post
title:  "OpenStack Mitaka从零开始 openstack里的AMQP使用(3)"
date:   2017-04-01 12:50:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}


我们先看MessageHandlingServer类,也就是RPC server,服务原型是l3-agent

```python

class MessageHandlingServer(service.ServiceBase, _OrderedTaskRunner):

    # 在RPCDispatcher一节我们已经大致知道MessageHandlingServer的功能了
    # 现在我们来详细看看MessageHandlingServer的实现
    # 从继承ServiceBase可以看出,start里肯定有死循环

    def __init__(self, transport, dispatcher, executor='blocking'):
        self.conf = transport.conf
        self.conf.register_opts(_pool_opts)
        self.transport = transport
        self.dispatcher = dispatcher
        self.executor_type = executor
        self.listener = None
        # 这就是在setuptool的entry_points.txt文件中找到
        # eventlet = futurist:GreenThreadPoolExecutor
        # GreenThreadPoolExecutor in module futurist._futures:
        try:
            mgr = driver.DriverManager('oslo.messaging.executors',
                                       self.executor_type)
        except RuntimeError as ex:
            raise ExecutorLoadFailure(self.executor_type, ex)
        # _executor_cls就是GreenThreadPoolExecutor类
        self._executor_cls = mgr.driver
        self._work_executor = None
        self._poll_executor = None
        self._started = False
        super(MessageHandlingServer, self).__init__()

    @ordered(reset_after='stop')
    def start(self, override_pool_size=None):
        if self._started:
            LOG.warning(_LW('Restarting a MessageHandlingServer is inherently '
                            'racy. It is deprecated, and will become a noop '
                            'in a future release of oslo.messaging. If you '
                            'need to restart MessageHandlingServer you should '
                            'instantiate a new object.'))
        self._started = True

        # 调用监听,最终调用的是rabbitmq的驱动的listen方法的返回值
        try:
            self.listener = self.dispatcher._listen(self.transport)
        except driver_base.TransportDriverError as ex:
            raise ServerListenError(self.target, ex)

        executor_opts = {}

        # 用线程方式执行
        if self.executor_type == "threading":
            executor_opts["max_workers"] = (
                override_pool_size or self.conf.executor_thread_pool_size
            )
        # 绿色线程方式执行,我们看的是这个
        elif self.executor_type == "eventlet":
            # 绿化thread模块,使用eventlet必须通过patch方式绿化thread模块
            eventletutils.warn_eventlet_not_patched(
                expected_patched_modules=['thread'],
                what="the 'oslo.messaging eventlet executor'")
            # 参数设置
            executor_opts["max_workers"] = (
                override_pool_size or self.conf.executor_thread_pool_size
            )
        # 当我们用eventlet的方式的时候
        # 这里相当于生成两个绿色线程池实例
        # 但是这个绿色线程池不是eventlet封装的,是GreenThreadPoolExecutor封装的
        # 不使用eventlet的原因在于....兼容用了threading的代码
        # GreenThreadPoolExecutor比较复杂,专门一节说明
        # 参考http://www.lolizeppelin.com/2017/04/01/python-GreenThreadPoolExecutor/
        self._work_executor = self._executor_cls(**executor_opts)
        self._poll_executor = self._executor_cls(**executor_opts)
        # 调用的是 GreenThreadPoolExecutor的submit
        # 所以具体死循环就在self._runner中
        # 我们先别管是如何绿化的,直接看_runner函数就好
        return lambda: self._poll_executor.submit(self._runner)

    # 这部分是如何绿化的需要看懂GreenThreadPoolExecutor
    # 这个装饰器是专门catch错误的
    @excutils.forever_retry_uncaught_exceptions
    def _runner(self):
        # 这里是工作循环
        while self._started:
            # 这里是从rabbitmq中取出数据
            # listener是AMQPListener
            # 接下来我们要看AMQPListener的poll方法
            incoming = self.listener.poll(
                timeout=self.dispatcher.batch_timeout,
                prefetch_size=self.dispatcher.batch_size)
            if incoming:
                # 调用self._submit_work处理
                # self.dispatcher(incoming)的返回值
                # self.dispatcher(incoming)也就是
                # RPCDispatcher.__call__方法调用的位置
                # 返回值就是DispatcherExecutorContext类实例
                self._submit_work(self.dispatcher(incoming))
        # 走到这里表示调用过了stop
        # 退出前处理incoming中剩余的数据
        while True:
            incoming = self.listener.poll(
                timeout=self.dispatcher.batch_timeout,
                prefetch_size=self.dispatcher.batch_size)

            if incoming:
                self._submit_work(self.dispatcher(incoming))
            else:
                return

    def _submit_work(self, callback):
        # callback就是DispatcherExecutorContext类实例
        # 处理过程又涉及到了GreenThreadPoolExecutor
        # 我们直接理解为异步执行了
        # DispatcherExecutorContext中的run和done就好
        fut = self._work_executor.submit(callback.run)
        fut.add_done_callback(lambda f: callback.done())
    ......
```


incoming的结构和,请看[下一节](http://www.lolizeppelin.com/2017/04/05/openstack-AMQP-4/)
