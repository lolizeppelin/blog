---
layout: post
title:  "OpenStack Mitaka从零开始 openstack里的AMPQ使用(1)"
date:   2017-04-01 12:50:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}

先回顾下l3的启动部分

```python
# neutron.service.Service
class Service(n_rpc.Service):
    def __init__(self, host, binary, topic, manager, report_interval=None,
                 periodic_interval=None, periodic_fuzzy_delay=None,
                 *args, **kwargs):
        .....
        self.manager_class_name = manager
        manager_class = importutils.import_class(self.manager_class_name)
        # manager = neutron.agent.l3.agent.L3NATAgent
        # 这里的manager将作为endpoint传给后面的
        self.manager = manager_class(host=host, *args, **kwargs)
        .....
        # topic=topics.L3_AGENT,也就是字符串"l3_agent"
        super(Service, self).__init__(host, topic, manager=self.manager)

    def start(self):
        # 这个只有dhcp agent有用,其他都是空
        self.manager.init_host()
        # n_rpc.Service.start(),这里就是初始化rpc服务器的地方
        super(Service, self).start()
        ....
        self.manager.after_start()

```

n_rpc是# neutron.common.rpc

```python
# neutron.common.rpc.Service
class Service(service.Service):

    def __init__(self, host, topic, manager=None, serializer=None):
        # service.Service主要就是初始化了一个绿色线程池组
        # 也就是多了一个属性
        # self.tg = threadgroup.ThreadGroup(threads)
        super(Service, self).__init__()
        self.host = host
        # topic是字符串l3_agent
        self.topic = topic
        self.serializer = serializer
        # manager = neutron.agent.l3.agent.L3NATAgent
        if manager is None:
            self.manager = self
        else:
            self.manager = manager

    def start(self):
        # 这行没用
        super(Service, self).start()
        # 创建一个connection对象,注意这个Connection对象不是pika的Connection
        self.conn = create_connection()
        LOG.debug("Creating Consumer connection for Service %s",
                  self.topic)

        endpoints = [self.manager]
        # 作为endpoints传入给create_consumer
        # 当接收到消息队列数据的时候,最终调用endponrt中的函数
        # 这里是创建了MessageHandlingServer类并插入到conn.servers列表中
        # l3-agent只有一个MessageHandlingServer实例
        self.conn.create_consumer(self.topic, endpoints)

        # manager里有钩子函数,将当前Service实例注册到里面
        if callable(getattr(self.manager, 'initialize_service_hook', None)):
            self.manager.initialize_service_hook(self)
        # 调用conn.servers列表中的实例的start函数
        # 也就是调用MessageHandlingServer的start函数
        self.conn.consume_in_threads()

# 这个就是create_connection返回的Connection对象的类
class Connection(object):
    def __init__(self):
        super(Connection, self).__init__()
        # 可以看出这个Connection用于存放多个消息队列服务器
        self.servers = []

    def create_consumer(self, topic, endpoints, fanout=False):
        # oslo_messaging.Target类实例下面有说明
        # server的取值是默认是当前机器的hostname
        # topic是字符串l3_agent
        target = oslo_messaging.Target(
            topic=topic, server=cfg.CONF.host, fanout=fanout)
        # 每次调用create_consumer都会生成一个server
        # 这个server是MessageHandlingServer
        # 也就是rpc server, 就是一个基于ampq的rpc server
        # rpc server从ampq收到消息,调用endpoints执行继续的函数
        # server通过target和endpoints生成
        server = get_server(target, endpoints)
        self.servers.append(server)

    def consume_in_threads(self):
        for server in self.servers:
            server.start()
        return self.servers

    def close(self):
        for server in self.servers:
            server.stop()
        for server in self.servers:
            server.wait()

# 这个就是Target类
class Target(object):
    def __init__(self, exchange=None, topic=None, namespace=None,
                 version=None, server=None, fanout=None,
                 legacy_namespaces=None):
        # 从属性可以看出Target相当于一份配置文件
        # -----------------
        # 当Target的使用者是server的时候
        # topic和server是必要属性
        # exchange可选属性
        # -----------------
        # 与endpoint相关的namespace和version是可选属性
        # -----------------
        # 当Target的使用者是客户端的时候
        # 客户端发送信息, topic是必须的,其他参数都是可选的
        # -----------------
        # 注意,neutron server是发送方、是客户端
        # 各个agent是接收方、是服务端
        self.exchange = exchange
        self.topic = topic
        self.namespace = namespace
        self.version = version
        # Clients can request that a message be directed to a specific
        # server, rather than just one of a pool of servers listening on the topic.
        # 客户端从指定的server请求信息而不是从监听topic的的servers组成的池中选择
        self.server = server
        self.fanout = fanout
        self.accepted_namespaces = [namespace] + (legacy_namespaces or [])

    def __call__(self, **kwargs):
        # 可以看出这里复制了一份自身
        for a in ('exchange', 'topic', 'namespace',
                  'version', 'server', 'fanout'):
            kwargs.setdefault(a, getattr(self, a))
        return Target(**kwargs)

    ...
```

```python

def get_server(target, endpoints, serializer=None):
    # TRANSPORT就是rabbitmq的驱动
    # 看后面TRANSPORT的代码
    assert TRANSPORT is not None
    # serializer是RequestContextSerializer
    serializer = RequestContextSerializer(serializer)
    return oslo_messaging.get_rpc_server(TRANSPORT, target, endpoints,
                                         'eventlet', serializer)

def get_rpc_server(transport, target, endpoints,
                executor='blocking', serializer=None):
     """Construct an RPC server.
     从ampq到来的消息如何被接收和分发由executor来控制,最简单的executor是阻塞模式
     如果使用eventlet作为executor,threading和time模块必须被monkeypatch过(默认使用eventlet)

     :param transport: the messaging transport
     :param target: the exchange, topic and server to listen on
     :param endpoints: 是manager(在l3 agent中就是L3NATAgent)组成的列表
     :type executor: 指定为eventlet字符串
     :param serializer: 用于将amqp数据序列化
     """
     # dispatcher就是分发器
     dispatcher = rpc_dispatcher.RPCDispatcher(target, endpoints, serializer)
     # MessageHandlingServer就是rpc server
     return msg_server.MessageHandlingServer(transport, dispatcher, executor)
```

从上面我们可以看出,rpc server,也就是MessageHandlingServer实例的两个主要参数

1、dispatcher
2、transport


我们先来看TRANSPORT的初始化,TRANSPORT是一个全局变量

```python
def init(conf):
    global TRANSPORT, NOTIFICATION_TRANSPORT, NOTIFIER
    exmods = get_allowed_exmods()
    # 初始化TRANSPORT
    TRANSPORT = oslo_messaging.get_transport(conf,
                                             allowed_remote_exmods=exmods,
                                             aliases=TRANSPORT_ALIASES)
    # 初始化NOTIFICATION_TRANSPORT
    # get_notification_transport是
    # 读取oslo_messaging_notifications部分配置文件的get_transport
    NOTIFICATION_TRANSPORT = oslo_messaging.get_notification_transport(
        conf, allowed_remote_exmods=exmods, aliases=TRANSPORT_ALIASES)
    serializer = RequestContextSerializer()
    # NOTIFIER就是NOTIFICATION_TRANSPORT的再封装
    # 外部直接调用NOTIFIER来发送notification而不是用NOTIFICATION_TRANSPORT
    # NOTIFICATION_TRANSPORT和TRANSPORT为什么要拆分是因为
    # TRANSPORT要封装rpc server,是rpc服务
    # NOTIFICATION_TRANSPORT要封装为NOTIFIER, 只发消息
    # 这次我们重点在TRANSPORT,所以NOTIFIER相关的东西略过
    NOTIFIER = oslo_messaging.Notifier(NOTIFICATION_TRANSPORT,
                                       serializer=serializer)

# TRANSPORT的创建函数get_transport
# get_notification_transport最终也是调用get_transport
def get_transport(conf, url=None, allowed_remote_exmods=None, aliases=None):
    ......
    # 初始化rabbit驱动,rabbit驱动,前面省略校验代码
    try:
       mgr = driver.DriverManager('oslo.messaging.drivers',
                                  url.transport.split('+')[0],
                                  invoke_on_load=True,
                                  invoke_args=[conf, url],
                                  invoke_kwds=kwargs)
    except RuntimeError as ex:
       raise DriverLoadFailure(url.transport, ex)

    return Transport(mgr.driver)


class Transport(object):
    # 代码可以看出Transport是直接封装了rabbit驱动的调用
    # 也就是信息的收发通过Transport
    # 里面没有直接的接收方法
    # 都是通过liensn注册到驱动中接收数据
    def __init__(self, driver):
        self.conf = driver.conf
        self._driver = driver

    def cleanup(self):
        """Release all resources associated with this transport."""
        self._driver.cleanup()

    def _require_driver_features(self, requeue=False):
        self._driver.require_features(requeue=requeue)

    def _send(self, target, ctxt, message, wait_for_reply=None, timeout=None,
              retry=None):
        # 调用send, target必须有topic属性
        if not target.topic:
            raise exceptions.InvalidTarget('A topic is required to send',
                                           target)
        return self._driver.send(target, ctxt, message,
                                 wait_for_reply=wait_for_reply,
                                 timeout=timeout, retry=retry)
    def _listen(self, target):
        # 调用_listen, 必须同时有topic、和server属性
        if not (target.topic and target.server):
         raise exceptions.InvalidTarget('A server\'s target must have '
                                        'topic and server names specified',
                                        target)
        # 这里初始化rabbit驱动的监听
        return self._driver.listen(target)

    # ------------------下面的函数是NOTIFIER用的-----------------------
    # NOTIFIER发信息调用_send_notification
    def _send_notification(self, target, ctxt, message, version, retry=None):
        # 调用_send_notification, target必须有topic属性
        if not target.topic:
            raise exceptions.InvalidTarget('A topic is required to send',
                                           target)
        self._driver.send_notification(target, ctxt, message, version,
                                       retry=retry)


    # NOTIFIER接收信息调用_listen_for_notifications来监听
    def _listen_for_notifications(self, targets_and_priorities, pool):
        # 这个函数应该是notification直接用了内部消息组件的是后调用的
        for target, priority in targets_and_priorities:
            if not target.topic:
                raise exceptions.InvalidTarget('A target must have '
                                               'topic specified',
                                               target)
        return self._driver.listen_for_notifications(
            targets_and_priorities, pool)

```


RPCDispatcher比较复杂,我们在下一节接着讲
