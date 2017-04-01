---
layout: post
title:  "OpenStack Mitaka从零开始 openstack里的AMPQ使用(3)"
date:   2017-04-01 12:50:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}


incoming的先放一下先看MessageHandlingServer类


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

        # _executor_cls就是GreenThreadPoolExecutor
        self._executor_cls = mgr.driver

        self._work_executor = None
        self._poll_executor = None

        self._started = False

        super(MessageHandlingServer, self).__init__()

    def _submit_work(self, callback):
        fut = self._work_executor.submit(callback.run)
        fut.add_done_callback(lambda f: callback.done())

    @ordered(reset_after='stop')
    def start(self, override_pool_size=None):
        if self._started:
            LOG.warning(_LW('Restarting a MessageHandlingServer is inherently '
                            'racy. It is deprecated, and will become a noop '
                            'in a future release of oslo.messaging. If you '
                            'need to restart MessageHandlingServer you should '
                            'instantiate a new object.'))
        self._started = True

        # 调用监听,最终调用的是rabbitmq的驱动的listen方法
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

        # GreenThreadPoolExecutor比较复杂,专门一节说明
        self._work_executor = self._executor_cls(**executor_opts)
        self._poll_executor = self._executor_cls(**executor_opts)

        return lambda: self._poll_executor.submit(self._runner)

    @ordered(after='start')
    def stop(self):
        """Stop handling incoming messages.

        Once this method returns, no new incoming messages will be handled by
        the server. However, the server may still be in the process of handling
        some messages, and underlying driver resources associated to this
        server are still in use. See 'wait' for more details.
        """
        self.listener.stop()
        self._started = False

    @excutils.forever_retry_uncaught_exceptions
    def _runner(self):
        while self._started:
            incoming = self.listener.poll(
                timeout=self.dispatcher.batch_timeout,
                prefetch_size=self.dispatcher.batch_size)

            if incoming:
                self._submit_work(self.dispatcher(incoming))

        # listener is stopped but we need to process all already consumed
        # messages
        while True:
            incoming = self.listener.poll(
                timeout=self.dispatcher.batch_timeout,
                prefetch_size=self.dispatcher.batch_size)

            if incoming:
                self._submit_work(self.dispatcher(incoming))
            else:
                return

    @ordered(after='stop')
    def wait(self):
        """Wait for message processing to complete.

        After calling stop(), there may still be some existing messages
        which have not been completely processed. The wait() method blocks
        until all message processing has completed.

        Once it's finished, the underlying driver resources associated to this
        server are released (like closing useless network connections).
        """
        self._poll_executor.shutdown(wait=True)
        self._work_executor.shutdown(wait=True)

        # Close listener connection after processing all messages
        self.listener.cleanup()

    def reset(self):
        """Reset service.

        Called in case service running in daemon mode receives SIGHUP.
        """
        # TODO(sergey.vilgelm): implement this method
        pass



```
