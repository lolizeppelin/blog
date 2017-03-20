---
layout: post
title:  "OpenStack Mitaka从零开始 openstack里eventlet的使用"
date:   2017-03-13 12:50:00 +0800
categories: "虚拟化"
tag: ["openstack", "python", "linux"]
---

* content
{:toc}

openstack里如何使用monkey patch的参考[这篇](http://www.lolizeppelin.com/2017/03/13/python-modules-init/)

然后我们要学习[eventlet使用,绿色线程工作原理](http://www.lolizeppelin.com/2017/03/10/python-eventlet/)

看完绿色线程工作原理我们来看看L3 Agent的入口start部分,找到绿色线程的入口代码

```python

# 回顾下前面的代码
# 子进程中
# _start_child->_child_process->launch_service->services.add

class Services
    def __init__(self):
        self.services = []
        self.tg = threadgroup.ThreadGroup()
        # event对象,这里我们看到Services中有event对象
        self.done = event.Event()
     ....
     def add(self, service):
         .....
         self.tg.add_thread(self.run_service, service, self.done)

    @staticmethod
    def run_service(service, done):
        # 这里调用了service的start方法并wait
        # done是Services类实力的done属性
        # done是一个Event.event()
        try:
            service.start()
        except Exception:
            LOG.exception(_LE('Error starting thread.'))
            raise SystemExit(1)
        else:
            # 代码一直不会走到这里
            # 直到service.start()结束
            done.wait()

    def stop(self):
        .....
        # 可以看到这里调用了self.done.send()
        # 这样wait就能获取到结果然后结束
        if not self.done.ready():
            self.done.send()
        ....

# Service类的初始化
server = neutron_service.Service.create(
    binary='neutron-l3-agent',
    topic=topics.L3_AGENT,
    report_interval=cfg.CONF.AGENT.report_interval,
    manager=manager)

# 所以我们的Service类为
class Service(n_rpc.Service):
    ...
    # 这里也就是run_service函数
    # 最终调用的service.start的位置
    def start(self):
        ...
        # 省略非关键代码
        #  n_rpc.Service中的代码合并过来
        super(Service, self).start()
        # socket代码经过monkey path
        # 所有的cocket的recv  send connect代码都是绿化过的
        self.conn = create_connection()
        LOG.debug("Creating Consumer connection for Service %s",
                  self.topic)
        # 关键点!!!!
        # self.manager是L3NATAgent
        # L3NATAgent作为endpoints传到rpc接口   
        # 接受到rabbit的数据包的时候
        # 自动会调用endpoints,也就是L3NATAgent实例来处理数据包   
        endpoints = [self.manager]
        self.conn.create_consumer(self.topic, endpoints)
        if callable(getattr(self.manager, 'initialize_service_hook', None)):
            self.manager.initialize_service_hook(self)
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
        # 生成一个绿色线程池,只有8个绿色线程
        pool = eventlet.GreenPool(size=8)
        # 因为只有8个绿色线程,所以这里会在
        # 孵化8个_process_router_update绿色线程后被阻塞
        # 阻塞形式是以不停的切换到main loop中的形势阻塞
        # 并不是真正的卡住阻塞
        # 这里写永真循环的作用在于当任意一个_process_router_update绿色线程挂了
        # 都能再启一个_process_router_update绿色线程
        while True:
            pool.spawn_n(self._process_router_update)

```
