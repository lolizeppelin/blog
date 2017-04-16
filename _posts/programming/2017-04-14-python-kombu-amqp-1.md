---
layout: post
title:  "python kombu与amqp(1)"
date:   2017-04-14 12:50:00 +0800
categories: "编程"
tag: ["python", "linux"]
---

* content
{:toc}


首先搞清楚pika、kombu、AMQP的关系

AMQP

    AMQP，即Advanced Message Queuing Protocol,一个提供统一消息服务的应用层标准
    高级消息队列协议,是应用层协议的一个开放标准,为面向消息的中间件设计。
    基于此协议的客户端与消息中间件可传递消息，并不受客户端/中间件不同产品，
    不同的开发语言等条件的限制。常见的消息中间件有RabbitMQ、ActiveMQ和ZeroMQ

    RabbitMQ是AMQP协议领先的一个实现，它实现了代理(Broker)架构，意味着消息在发送到客户端
    之前可以在中央节点上排队。此特性使得RabbitMQ易于使用和部署，适宜于很多场景如路由、
    负载均衡或消息持久化等，用消息队列只需几行代码即可搞定。
    但是，这使得它的可扩展性差，速度较慢，因为中央节点增加了延迟，消息封装后也比较大。
    openstack默认使用RabbitMQ


    ZeroMQ是一个非常轻量级的消息系统，专门为高吞吐量/低延迟的场景开发，
    在金融界的应用中经常可以发现它。与RabbitMQ相比，ZeroMQ支持许多高级消息场景，
    但是你必须实现ZeroMQ框架中的各个块（比如Socket或Device等）。
    ZeroMQ非常灵活，但是你必须学习它的80页的手册（如果你要写一个分布式系统，一定要阅读它）
    saltstack使用ZeroMQ


    ActiveMQ居于两者之间，类似于ZemoMQ，它可以部署于代理模式和P2P模式。
    类似于RabbitMQ，它易于实现高级场景，而且只需付出低消耗。它被誉为消息中间件的“瑞士军刀”
    要注意一点，ActiveMQ的下一代产品为Apollo。
    以前java用这个比较多,现在不清楚

    其他还有HornetQ, QPID、kafka等

Pika

    Pika is a pure-Python implementation of the AMQP 0-9-1
    pika是专门用来实现AMQP 0-9-1规范的库

kombu

    Kombu is a messaging library for Python.
    kombu是一个消息库,支持RabbitMQ、AMQP、py-amqp、Zookeeper等多种
    与Pika相比,kombu是一个大杂烩,而Pika只支持AMQP 0-9-1规范
    kombu处理AMQP协议没有直接用Pika,而是自己写了一套
    kombu项目变大以后,把处理AMQP协议的部分独立成了一个项目py-amqp,在python里包名amqp
    kombu对rabbitmq可以用c写的librabbitmq,不过librabbitmq不支持eventlet
    所以在openstack里,kombu还是用amqp来和rabbitmq通信

因为kombu支持的类型多,甚至可以用redis来模拟消息队列,现在openstack默认使用kombu


因为pika相对简单,我们先通过pika来简单看看rabbitmq的使用


```python
import pika
from pika import PlainCredentials
from pika import ConnectionParameters

# 发送者
def publish():
    user = 'phptest'
    passwd = 'phptest'
    vhost = 'phptest'
    ip = '127.0.0.1'
    port = 5673
    identified = PlainCredentials(user, passwd)
    paarameters = ConnectionParameters(ip, port, vhost, identified)
    # BlockingConnection类就是一个rabbitmq的连接,如果要用多连接的话,pika有一个pika-pool的库
    connection  = pika.BlockingConnection(paarameters)
    # channel这个玩意比较蛋碎,之前我看了很久就是为了看明白为什么不直接用connection
    # 还要在connection上封一层channel,后来大致明白
    # 其实就是为了不多建立多个connection也能做隔离
    channel = connection.channel()
    print 'connect success'
    # 声明一个叫gcy2的交换机(exchange), 默认的exchange_type是direct,即单点的
    channel.exchange_declare(exchange='gcy2', auto_delete=True)
    print 'start send data'
    # 然后发送数据 到gcy2 这个exchange, 接收者的routing_key是a
    channel.basic_publish(exchange='gcy2',routing_key='a', body='wtffffff1')
    channel.basic_publish(exchange='gcy2',routing_key='a', body='wtffffff2')
    print 'end'

# 接收者
def consume():
    user = 'phptest'
    passwd = 'phptest'
    vhost = 'phptest'
    ip = '127.0.0.1'
    port = 5673
    identified = PlainCredentials(user, passwd)
    paarameters = ConnectionParameters(ip, port, vhost, identified)
    connection  = pika.BlockingConnection(paarameters)
    channel = connection.channel()
    print 'connect success'
    # 声明一个队列
    channel.queue_declare(queue='myqeueu')
    # 队列绑定到交换机
    channel.queue_bind(queue='myqeueu', exchange='gcy2', routing_key='a')
    # 数据写入队列
    get_list = []
    # 回调函数
    def callback(ch, method, properties, body):
        print 'get body %s ' % body,
        print method.consumer_tag
        get_list.append([method.delivery_tag, body])
        # 回复ack
        ch.basic_ack(method.delivery_tag)
        #ch.stop_consuming()
    channel.basic_qos(prefetch_count=1)
    # 不自动回复ack,在callback中进行
    tag1 = channel.basic_consume(callback, queue='myqeueu', no_ack=False)
    # 循环事件
    # 演示代码没有循环只执行一次
    connection.process_data_events()
    connection.close()
```


简单是使用看完后,我们来看openstack里对kombu的使用

```python
# 初始化kombu的connection
self.connection = kombu.connection.Connection(
    self._url, ssl=self._fetch_ssl_params(),
    login_method=self.login_method,
    heartbeat=self.heartbeat_timeout_threshold,
    failover_strategy=self.kombu_failover_strategy,
    transport_options={
        'confirm_publish': True,
        'client_properties': {'capabilities': {
            'authentication_failure_close': True,
            'connection.blocked': True,
            'consumer_cancel_notify': True}},
        'on_blocked': self._on_connection_blocked,
        'on_unblocked': self._on_connection_unblocked,
    },
)

# 初始化队列
self.queue = kombu.entity.Queue(
    name=self.queue_name,
    channel=conn.channel,
    exchange=self.exchange,
    durable=self.durable,
    auto_delete=self.queue_auto_delete,
    routing_key=self.routing_key,
    queue_arguments=self.queue_arguments)
# 声明队列
self.queue.declare()

# 消费者声明
self.queue.consume(callback=self._callback,
                   consumer_tag=six.text_type(tag),
                   nowait=self.nowait)

# 自动重试
autoretry_method = self.connection.autoretry(
    execute_method, channel=self.channel,
    max_retries=retry,
    errback=on_error,
    interval_start=self.interval_start or 1,
    interval_step=self.interval_stepping,
    interval_max=self.interval_max,
    on_revive=on_reconnection)
# autoretry_method返回channel对象
ret, channel = autoretry_method()
# 设置channel, channel在这里被设置
self._set_current_channel(channel)
return ret
```


现在看看kombu中的Connection的初始化和队列各种声明都用到的autoretry


```python
# kombu.connection.Connection
class Connection(object):

     def __init__(self,....):
         # init主要处理了下参数
         .....

     # 外部通过default_channel来获取connectio的channel
     # 来看看default_channel是如何初始化的
     @property
     def default_channel(self):
         # 确认connection初始化
         self.connection
         if self._default_channel is None:
             # 生成channel
             self._default_channel = self.channel()
         return self._default_channel

     @property
     def connection(self):
         if not self._closed:
             if not self.connected:
                 self.declared_entities.clear()
                 self._default_channel = None
                 # 调用connection的初始化
                 self._connection = self._establish_connection()
                 self._closed = False
             return self._connection

     def _establish_connection(self):
         self._debug('establishing connection...')
         # 调用transport创建connection
         # transport就是amqp里的transport
         conn = self.transport.establish_connection()
         self._debug('connection established: %r', self)
         return conn

     def channel(self):
         self._debug('create channel')
         # 调用transport创建channel
         # transport就是amqp里的transport
         chan = self.transport.create_channel(self.connection)
         if _log_channel:  # pragma: no cover
             from .utils.debug import Logwrapped
             return Logwrapped(chan, 'kombu.channel',
                               '[Kombu channel:{0.channel_id}] ')
         return chan

     @property
     def transport(self):
         if self._transport is None:
             self._transport = self.create_transport()
         return self._transport

     def create_transport(self):
         # 注意参数client=self
         return self.get_transport_cls()(client=self)

     def get_transport_cls(self):
         transport_cls = self.transport_cls
         # transport_cls是字符串amqp
         if not transport_cls or isinstance(transport_cls, string_t):
             transport_cls = get_transport_cls(transport_cls)
         # transport_cls是kombu.transport.pyamqp.Transport
         # 下一节再看Transport
         return transport_cls

# -----------------下面是autoretry的实现---------------------

    def autoretry(self, fun, channel=None, **ensure_options):
        # 这里把channel变成channels列表
        # 不用列表会出现UnboundLocalError
        # 用列表原因参考闭包自由变量重新绑定
        channels = [channel]
        class Revival(object):
            __name__ = getattr(fun, '__name__', None)
            __module__ = getattr(fun, '__module__', None)
            __doc__ = getattr(fun, '__doc__', None)

            def __init__(self, connection):
                self.connection = connection

            def revive(self, channel):
                # 重新绑定自由变量
                channels[0] = channel

            def __call__(self, *args, **kwargs):
                if channels[0] is None:
                    # 没有channel,调用default_channel使用默认channel
                    self.revive(self.connection.default_channel)
                # 返回值是fun的返回值和channel
                return fun(*args, channel=channels[0], **kwargs), channels[0]
        # Revival用来实现闭包函数的类,自由变量是channels,闭包是revive
        revive = Revival(self)
        return self.ensure(revive, revive, **ensure_options)

    # 来看看ensure
    def ensure(self, obj, fun, errback=None, max_retries=None,
               interval_start=1, interval_step=1, interval_max=1,
               on_revive=None):

        def _ensured(*args, **kwargs):
            got_connection = 0
            conn_errors = self.recoverable_connection_errors
            chan_errors = self.recoverable_channel_errors
            has_modern_errors = hasattr(
                self.transport, 'recoverable_connection_errors',
            )
            with self._reraise_as_library_errors():
                # 这个写发好....不用定义i 然后i+=1了
                # 学习了
                # 一个永真循环, 这个循环应该是有上限的,上限应该是python int的最大值
                for retries in count(0):  # for infinity
                    try:
                        # 这个fun就是Revival.__call___
                        return fun(*args, **kwargs)
                    except conn_errors as exc:
                        # 成功获取过链接, transport有可接受的链接错误属性
                        if got_connection and not has_modern_errors:
                            # transport can not distinguish between
                            # recoverable/irrecoverable errors, so we propagate
                            # the error if it persists after a new connection
                            # was successfully established.
                            raise
                        # 超过重试次数
                        if max_retries is not None and retries > max_retries:
                            raise
                        self._debug('ensure connection error: %r',
                                    exc, exc_info=1)
                        self.collect()
                        # 有errback,调用errback
                        errback and errback(exc, 0)
                        remaining_retries = None
                        if max_retries is not None:
                            remaining_retries = max(max_retries - retries, 1)
                        # 这个是封装过的self.connection调用
                        # 用于确认链接
                        self.ensure_connection(
                            errback,
                            remaining_retries,
                            interval_start, interval_step, interval_max,
                            reraise_as_library_errors=False,
                        )
                        # 重设channel
                        channel = self.default_channel
                        obj.revive(channel)
                        if on_revive:
                            on_revive(channel)
                        # 成功获取到链接的计数器
                        got_connection += 1
                    except chan_errors as exc:
                        if max_retries is not None and retries > max_retries:
                            raise
                        self._debug('ensure channel error: %r',
                                    exc, exc_info=1)
                        errback and errback(exc, 0)
        _ensured.__name__ = bytes_if_py2('{0}(ensured)'.format(fun.__name__))
        _ensured.__doc__ = fun.__doc__
        _ensured.__module__ = fun.__module__
        # _ensured就是autoretry_method
        # autoretry_method()就是_ensured()
        # 返回值是Revival.__call___的返回值
        return _ensured
```

下一节我们看amqp包中的Transport,socket循环和messag的封装都在其中
