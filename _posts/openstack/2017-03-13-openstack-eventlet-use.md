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

class Services
     ....

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
            done.wait()


server = neutron_service.Service.create(
    binary='neutron-l3-agent',
    topic=topics.L3_AGENT,
    report_interval=cfg.CONF.AGENT.report_interval,
    manager=manager)

class Service(n_rpc.Service):
    ...
    # 这里也就是run_service函数
    # 最终调用的service.start的位置
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
