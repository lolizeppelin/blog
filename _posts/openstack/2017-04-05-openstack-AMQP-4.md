---
layout: post
title:  "OpenStack Mitaka从零开始 openstack里的AMQP使用(4)"
date:   2017-04-05 12:50:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}


现在我们可以来看incoming是什么了,首先我们回顾下

```python

# MessageHandlingServer是通过如下方式获取到amqp过来的消息
incoming = self.listener.poll(
    timeout=self.dispatcher.batch_timeout,
    prefetch_size=self.dispatcher.batch_size)
# self.listener的初始化是
self.listener = self.dispatcher._listen(self.transport)
# dispatcher的_listen的返回内容是
return transport._listen(self._target)
# transport的listen返回内容是
return self._driver.listen(target)
# 我们配置的rpc_backend = rabbit, 那么_driver就是RabbitDriver
```

我们现在来看RabbitDriver

```python

class RabbitDriver(amqpdriver.AMQPDriverBase):
    def __init__(self, conf, url,
                 default_exchange=None,
                 allowed_remote_exmods=None):
        # 配置文件相关省略
        .....
        # 重试时间
        self.missing_destination_retry_timeout = (
            conf.oslo_messaging_rabbit.kombu_missing_consumer_retry_timeout)

        # 这个和rabbit的qos有关
        self.prefetch_size = (
            conf.oslo_messaging_rabbit.rabbit_qos_prefetch_count)
        # 连接池对象, Connection参数是一个class
        # 在这里是oslo_messaging._drivers.impl_rabbit.Connection
        # rpc_conn_pool_size是连接池大小
        connection_pool = pool.ConnectionPool(
            conf, conf.oslo_messaging_rabbit.rpc_conn_pool_size,
            url, Connection)
        # 其他方法在AMQPDriverBase中
        super(RabbitDriver, self).__init__(
            conf, url,
            connection_pool,
            default_exchange,
            allowed_remote_exmods
        )

    def require_features(self, requeue=True):
        pass

# 上面的ConnectionPool类
class ConnectionPool(Pool):
    """Class that implements a Pool of Connections."""
    def __init__(self, conf, rpc_conn_pool_size, url, connection_cls):
        # connection_cls就是oslo_messaging._drivers.impl_rabbit.Connection
        self.connection_cls = connection_cls
        self.conf = conf
        self.url = url
        super(ConnectionPool, self).__init__(rpc_conn_pool_size)
        self.reply_proxy = None
    # agent是监听类,调用create
    # 服务器的调用get, get在父类Pool中
    def create(self, purpose=None):
        if purpose is None:
            purpose = common.PURPOSE_SEND
        LOG.debug('Pool creating new connection')
        # create最终通过connection_cls创建并返回实例
        # connection_cls就是rabbit里的connection,也就是
        # oslo_messaging._drivers.impl_rabbit.Connection
        return self.connection_cls(self.conf, self.url, purpose)
    # 略过其他代码
    ....

# RabbitDriver的父类
class AMQPDriverBase(base.BaseDriver):
    # base.BaseDriver基本都是abc
    # 只有一个类属性prefetch_size = 0
    ...

    def _get_connection(self, purpose=rpc_common.PURPOSE_SEND):
        # 下面有ConnectionContext类说明
        # _get_connection返回的ConnectionContext对象分send和listen两种类型
        # purpose参数用于区分是send型还是listen型
        return rpc_common.ConnectionContext(self._connection_pool,
                                            purpose=purpose)

    def listen(self, target):
        # PURPOSE_LISTEN参数表示要获取listen型的connection
        conn = self._get_connection(rpc_common.PURPOSE_LISTEN)
        # listener是AMQPListener类
        # 创建的时候把自己也作为参数传了进去
        listener = AMQPListener(self, conn)
        # 注册一个type为topic的交换机
        # routing_key=topic=target.topic=topics.L3_AGENT="l3_agent"
        # queue_name=queue_name or topic 也就是"l3_agent"
        conn.declare_topic_consumer(exchange_name=self._get_exchange(target),
                                    topic=target.topic,
                                    callback=listener)
        # 和上面一样,只不过topic和queue_name变成了"l3_agent.host"
        # host在neutron的相关组件中是宿主机的hostname
        conn.declare_topic_consumer(exchange_name=self._get_exchange(target),
                                    topic='%s.%s' % (target.topic,
                                                     target.server),
                                    callback=listener)
        # 注册了一个广播型的交换机
        # exchange_name = '%s_fanout' % topic = "l3_agent_fanout"
        # queue_name = '%s_fanout_%s' % (topic, unique) # 后缀随机的queue_name
        # routing_key = topic = "l3_agent"
        conn.declare_fanout_consumer(target.topic, listener)
        # 返回的listener就是AMQPListener实例,后面我继续看AMQPListener类
        return listener

class ConnectionContext:
    # ConnectionContext其实有继承Connection类
    # 但是这个Connection类不是rabbit的那个Connection只是名字一样
    # 这个Connection只有一个close方法
    # 所以这里删除掉以免误解
    def __init__(self, connection_pool, purpose):
        """Create a new connection, or get one from the pool."""
        self.connection = None
        self.connection_pool = connection_pool
        # 判断是send类型还是listen类型
        pooled = purpose == PURPOSE_SEND
        # 这里看出ConnectionContext是从connection_pool中
        # 获取到的一个connection的封装实例, 赋值到self.connection
        # 这个connection就是rabbit的那个connction了
        # 也就是# oslo_messaging._drivers.impl_rabbit.Connection的实例
        # 所以ConnectionContext相当于把connction的封装成有上下文对象的实力
        # 这样可以使用with语法,结束的是后自动调用done
        # self.connction比较复杂,是amqp相关的封装,需要专门一节和amqp等一起讲
        # send型调用get返回connection实例
        if pooled:
            self.connection = connection_pool.get()
        # listen型调用create返回connection实例
        else:
            # agent是listen类型,走create方法,可以看前面connection_pool中对应实现
            self.connection = connection_pool.create(purpose)
        self.pooled = pooled
        self.connection.pooled = pooled

    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_value, tb):
        """End of 'with' statement.  We're done here."""
        self._done()

    def _done(self):
        # self.connection不为空
        if self.connection:
            # 如果是send型
            if self.pooled:
                # Reset the connection so it's ready for the next caller
                # to grab from the pool
                try:
                    # 重置链接,这里应该是标记链接可以被复用了
                    self.connection.reset()
                except Exception:
                    LOG.exception(_LE("Fail to reset the connection, drop it"))
                    try:
                        self.connection.close()
                    except Exception:
                        pass
                    self.connection = self.connection_pool.create()
                finally:
                    self.connection_pool.put(self.connection)
            # 监听型
            else:
                try:
                    # 这里并没有真正关闭链接
                    # 这里只是调用了connection的close()方法
                    # 而connection.close()并不关闭链接
                    # 具体看下一节
                    self.connection.close()
                except Exception:
                    pass
            self.connection = None
    # 其他属性都从connection里获取
    # 对ConnectionContext大部分方法的调用其实都是在调用
    # connection的对应方法
    def __getattr__(self, key):
        """Proxy all other calls to the Connection instance."""
        if self.connection:
            return getattr(self.connection, key)
        else:
            raise InvalidRPCConnectionReuse()
    .....
```

接下来看看AMQPListener类

```python

class AMQPListener(base.Listener):
    # base.Listener没有内容只有abc
    def __init__(self, driver, conn):
        # 可以看到两个参数,第一个参数是RabbitDriver实例
        # 第二个参数是ConnectionContext实例
        super(AMQPListener, self).__init__(driver.prefetch_size)
        self.driver = driver
        self.conn = conn
        self.msg_id_cache = rpc_amqp._MsgIdCache()
        self.incoming = []
        self._stopped = threading.Event()
        self._obsolete_reply_queues = ObsoleteReplyQueuesCache()

    def __call__(self, message):
        # AMQPDriverBase在declare的时候将AMQPListener实例callback传入
        # 也就是说__call__肯定是在rabbit的connection类中调用
        # 这里也就是apmq消息的入口
        # message是什么要看下一节
        ctxt = rpc_amqp.unpack_context(message)
        unique_id = self.msg_id_cache.check_duplicate_message(message)
        LOG.debug("received message msg_id: %(msg_id)s reply to %(queue)s", {
            'queue': ctxt.reply_q, 'msg_id': ctxt.msg_id})
        # 这里就incoming的类了
        # incoming是AMQPIncomingMessage类实例组成的列表
        # AMQPIncomingMessage相当于封装了一下message
        self.incoming.append(AMQPIncomingMessage(self,
                                                 ctxt.to_dict(),
                                                 message,
                                                 unique_id,
                                                 ctxt.msg_id,
                                                 ctxt.reply_q,
                                                 self._obsolete_reply_queues))
    # 这个装饰器处理了prefetch_size参数
    # 加入了超时定时器
    # 并返回列表
    @base.batch_poll_helper
    # pool就是rpc server的死循环中获取incoming的函数了
    # 这里就是消息的出口
    def poll(self, timeout=None):
        # 由MessageHandlingServer的_runner方法的死循环中反复调用
        while not self._stopped.is_set():
            if self.incoming:
                # 取出并返回列表中第一个元素
                # 对比__call__收到的数据总是插入到列表的最后
                # 这里返回的是AMQPIncomingMessage实例
                # 但是外部取的是incoming[0]
                # 因为这里通过batch_poll_helper装饰器
                # 把返回转成了只有一个值的列表
                # 看下面batch_poll_helper
                return self.incoming.pop(0)
            try:
                # 这里的consume不光是绑定消费者
                # 同时也调用数据接收函数也就是kombu里的drain_events
                # 如果消费者没有绑定过,先订阅消费者(将callback也就自身的传入)
                # 如过消费者已经订阅过队列了
                # 一旦有数据到来就会调用前面的__call__塞入amqp数据
                # 前面也就能pop出数据了
                # 这个循环实现了接受一个数据、处理一个数据、再收一个数据的循环
                self.conn.consume(timeout=timeout)
            except rpc_common.Timeout:
                return None

    def stop(self):
        self._stopped.set()
        self.conn.stop_consuming()

    def cleanup(self):
        # Closes listener connection
        self.conn.close()

def batch_poll_helper(func):
    def wrapper(in_self, timeout=None, prefetch_size=1):
        incomings = []
        driver_prefetch = in_self.prefetch_size
        # in_self就是AMQPListener实例
        # AMQPListener实例的prefetch_size就是
        # RabbitDriver实例的prefetch_size
        # 取值是rabbit_qos_prefetch_count,默认值是0
        if driver_prefetch > 0:
            # 取两者间最小值
            prefetch_size = min(prefetch_size, driver_prefetch)
        # 这是一个超时监视器,我们先不管具体是如何实现的
        watch = timeutils.StopWatch(duration=timeout)
        with watch:
            # MessageHandlingServer中pool的时候
            # 传入的prefetch_size是dispatcher.batch_size
            # 这个值是固定值1
            # 所以在MessageHandlingServer获取到的
            # incomings总是一个只有一个元素的列表
            for __ in compat_range(prefetch_size):
                # func是AMQPListener.pool
                msg = func(in_self, timeout=watch.leftover(return_none=True))
                if msg is not None:
                    incomings.append(msg)
                else:
                    # timeout reached or listener stopped
                    break
                time.sleep(0)
        return incomings
    return wrapper

# pool的incoming就是AMQPIncomingMessage实例
class AMQPIncomingMessage(base.RpcIncomingMessage):

    def __init__(self, listener, ctxt, message, unique_id, msg_id, reply_q,
                 obsolete_reply_queues):
        super(AMQPIncomingMessage, self).__init__(ctxt, message)
        # 父类的功能是设置两个属性
        # ctxt是message的ctxt属性转成的字典
        # message就是具体body
        self.ctxt = ctxt
        self.message = message
        # ---------------------
        # listener就是AMQPListener
        self.listener = listener
        # 下面两个参数是message和ctxt中的相关id
        self.unique_id = unique_id
        self.msg_id = msg_id
        # reply_q在AMQPListener中是
        # message的ctxt的reply_q方法
        self.reply_q = reply_q
        self._obsolete_reply_queues = obsolete_reply_queues
        # 一个超时监视器
        self.stopwatch = timeutils.StopWatch()
        self.stopwatch.start()


    def reply(self, reply=None, failure=None, log_failure=True):
        # 这里就是RPCDispatcher里调用的incoming.reply
        # reply就是RPCDispatcher落地操作的返回值
        if not self.msg_id:
            # NOTE(Alexei_987) not sending reply, if msg_id is empty
            #    because reply should not be expected by caller side
            return

        # NOTE(sileht): return without hold the a connection if possible
        if not self._obsolete_reply_queues.reply_q_valid(self.reply_q,
                                                         self.msg_id):
            return
        # 超时计数器
        duration = self.listener.driver.missing_destination_retry_timeout
        timer = rpc_common.DecayingTimer(duration=duration)
        timer.start()
        while True:
            try:
                # 调用的AMQPListener.driver_get_connection
                # 也就是RabbitDriver._get_connection
                # 返回的是ConnectionContext
                with self.listener.driver._get_connection(
                        rpc_common.PURPOSE_SEND) as conn:
                    # 通过_send_reply应答落地执行结果
                    # 第一个参数是get到的connection
                    # 第二个是落地操作返回值
                    self._send_reply(conn, reply, failure,
                                     log_failure=log_failure)
                return
            # 这个返回目的没找到,这个异常是由ConnectionPool类的get中抛出的
            except rpc_amqp.AMQPDestinationNotFound:
                # 还在超时时间内
                if timer.check_return() > 0:
                    LOG.debug(("The reply %(msg_id)s cannot be sent  "
                               "%(reply_q)s reply queue don't exist, "
                               "retrying..."), {
                                   'msg_id': self.msg_id,
                                   'reply_q': self.reply_q})
                    time.sleep(0.25)
                else:
                    self._obsolete_reply_queues.add(self.reply_q, self.msg_id)
                    LOG.info(_LI("The reply %(msg_id)s cannot be sent  "
                                 "%(reply_q)s reply queue don't exist after "
                                 "%(duration)s sec abandoning..."), {
                                     'msg_id': self.msg_id,
                                     'reply_q': self.reply_q,
                                     'duration': duration})
                    return


    def _send_reply(self, conn, reply=None, failure=None, log_failure=True):
        if not self._obsolete_reply_queues.reply_q_valid(self.reply_q,
                                                         self.msg_id):
            return

        if failure:
            failure = rpc_common.serialize_remote_exception(failure,
                                                            log_failure)
        # 将返回值reply封装成返回字典
        msg = {'result': reply, 'failure': failure, 'ending': True,
               '_msg_id': self.msg_id}
        rpc_amqp._add_unique_id(msg)
        unique_id = msg[rpc_amqp.UNIQUE_ID]

        LOG.debug("sending reply msg_id: %(msg_id)s "
                  "reply queue: %(reply_q)s "
                  "time elapsed: %(elapsed)ss", {
                      'msg_id': self.msg_id,
                      'unique_id': unique_id,
                      'reply_q': self.reply_q,
                      'elapsed': self.stopwatch.elapsed()})
        # 最终调用conn.direct_send,将落地操作返回值发送出去
        # reply_q是message的ctxt的reply_q方法
        # 那么这里其实就是从哪里来,发回给哪
        conn.direct_send(self.reply_q, rpc_common.serialize_msg(msg))

    def acknowledge(self):
        # rabbit mq回复ack
        self.message.acknowledge()
        self.listener.msg_id_cache.add(self.unique_id)

    def requeue(self):
        # NOTE(sileht): In case of the connection is lost between receiving the
        # message and requeing it, this requeue call fail
        # but because the message is not acknowledged and not added to the
        # msg_id_cache, the message will be reconsumed, the only difference is
        # the message stay at the beginning of the queue instead of moving to
        # the end.
        self.message.requeue()
```

message的结构现在还不明确,而message又是通过Connection传入的
```python
oslo_messaging._drivers.impl_rabbit.Connection
```
所以我们接下来要去看Connection类,请看[下一节](http://www.lolizeppelin.com/2017/04/07/openstack-AMQP-5/)
