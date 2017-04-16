---
layout: post
title:  "OpenStack Mitaka从零开始 openstack里的AMQP使用(5)"
date:   2017-04-07 12:50:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}

这节主要讲解Connection类以及相关的Consumer类,同时确认了AMQPListener.\__call__传入的变量是什么


```python

# oslo_messaging._drivers.impl_rabbit.Connection

class Connection(object):
    """Connection object."""

    pools = {}

    def __init__(self, conf, url, purpose):
        # NOTE(viktors): Parse config options
        driver_conf = conf.oslo_messaging_rabbit
        ......
        # 省略属性部分
        # 当前最大重试次数
        # 来源是配置文件的rabbit_max_retries
        # 用于控制ensure中的默认retry次数
        if self.max_retries <= 0:
            self.max_retries = None
        # rabbit的 vhost
        if url.virtual_host is not None:
            virtual_host = url.virtual_host
        else:
            virtual_host = self.virtual_host
        self._url = ''
        if self.fake_rabbit:
            # 假冒rabbit方式
            ....
        elif url.hosts:
            # 常规配置走这里
            if url.transport.startswith('kombu+'):
                LOG.warning(_LW('Selecting the kombu transport through the '
                                'transport url (%s) is a experimental feature '
                                'and this is not yet supported.'),
                            url.transport)
            if len(url.hosts) > 1:
                random.shuffle(url.hosts)
            for host in url.hosts:
                # 我们的rabbit走这里
                transport = url.transport.replace('kombu+', '')
                # rabbit被替换成amqp
                # 所以我们的最终驱动其实是amqp,和py-amqp有关
                transport = transport.replace('rabbit', 'amqp')
                # 用字符串";"合并url
                self._url += '%s%s://%s:%s@%s:%s/%s' % (
                    ";" if self._url else '',
                    transport,
                    parse.quote(host.username or ''),
                    parse.quote(host.password or ''),
                    self._parse_url_hostname(host.hostname) or '',
                    str(host.port or 5672),
                    virtual_host)
        elif url.transport.startswith('kombu+'):
            ....
        else:
            .....
        self._initial_pid = os.getpid()
        self._consumers = {}
        self._new_tags = set()
        self._active_tags = {}
        self._tags = itertools.count(1)
        self._consume_loop_stopped = False
        # 当前connection所在的channel
        self.channel = None
        self.purpose = purpose
        # send和listen的锁不一样
        if purpose == rpc_common.PURPOSE_SEND:
            self._connection_lock = ConnectionLock()
        else:
            self._connection_lock = DummyConnectionLock()
        # 初始化kombu的Connection
        # socket数据的收发,amqp消息的封装生成都在kombu里完成
        # kombu最终是调用py-amqp封装成amqp数据包的
        # 所以url里把rabbit替换成了amqp
        self.connection = kombu.connection.Connection(
            self._url, ssl=self._fetch_ssl_params(),
            # login_method没什么大用,不同登陆模式区别看
            # https://www.rabbitmq.com/authentication.html
            login_method=self.login_method,
            heartbeat=self.heartbeat_timeout_threshold,
            failover_strategy=self.kombu_failover_strategy,
            # 这个是py-amqp用的参数
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

        LOG.debug('Connecting to AMQP server on %(hostname)s:%(port)s',
                  self.connection.info())
        # 心跳相关
        # 单次心跳超时是心跳阈值时间/(心跳阈值时间内发送心跳次数/2)
        self._heartbeat_wait_timeout = (
            float(self.heartbeat_timeout_threshold) /
            float(self.heartbeat_rate) / 2.0)
        self._heartbeat_support_log_emitted = False
        # 这里是来时socket连接
        self.ensure_connection()
        # 心跳线程,当我们用的是eventlet的时候,走的也是绿色线程
        self._heartbeat_thread = None
        # 为send类型,启动心跳监听
        if purpose == rpc_common.PURPOSE_SEND:
            # 在这里生成线程赋值给_heartbeat_thread
            self._heartbeat_start()
        LOG.debug('Connected to AMQP server on %(hostname)s:%(port)s '
                  'via [%(transport)s] client',
                  self.connection.info())
        # 超时设置相关
        if self._heartbeat_supported_and_enabled():
            self._poll_timeout = self._heartbeat_wait_timeout
        else:
            self._poll_timeout = 1
        ....

    # 判断是否支持心跳
    def _heartbeat_supported_and_enabled(self):
        # 配置文件里心跳阈值<=0
        # 配置里默认值是60
        if self.heartbeat_timeout_threshold <= 0:
            return False
        # 这里是看kombu是否支持心跳
        if self.connection.supports_heartbeats:
            return True
        # 如果当前kombu版本的connection不支持心跳
        # 设置_heartbeat_support_log_emitted参数为true后再返回False
        # _heartbeat_support_log_emitted我没发现有什么用
        elif not self._heartbeat_support_log_emitted:
            LOG.warning(_LW("Heartbeat support requested but it is not "
                            "supported by the kombu driver or the broker"))
            self._heartbeat_support_log_emitted = True
        return False

    def ensure_connection(self):
        self._set_current_channel(None)
        # ensure比较复杂,method本身需要callable,所以这里直接弄了个匿名函数
        # 返回值是self.connection.connection
        # self.connection是kombu的connection
        # self.connection.connection就是amqp的connection
        self.ensure(method=lambda: self.connection.connection)
        self.set_transport_socket_timeout()

    def _set_current_channel(self, new_channel):
        # 设置当前正在使用的channel
        if new_channel == self.channel:
            return
        if self.channel is not None:
            self.PUBLISHER_DECLARED_QUEUES.pop(self.channel, None)
            self.connection.maybe_close_channel(self.channel)
        self.channel = new_channel
        if (new_channel is not None and
           self.purpose == rpc_common.PURPOSE_LISTEN):
           # 监听型设置qos,默认是没有qos限制
            self._set_qos(new_channel)

    def ensure(self, method, retry=None,
               recoverable_error_callback=None, error_callback=None,
               timeout_is_error=True):
        """
        底层执行函数,基本不会直接调用,被其他函数调用
        用于执行method,重试次数根据retry的值来定
        retry = None 相当于 retry=rabbit_max_retries
        retry = -1 则无限重试
        retry = 0 不重试
        retry = N 重试N次
        这个函数调用前必须调用connection lock
        ensure执行的method不能有参数
        """
        # 当前pid
        current_pid = os.getpid()
        # pid不是原始pid
        if self._initial_pid != current_pid:
            LOG.warning(_LW("Process forked after connection established! "
                            "This can result in unpredictable behavior. "
                            "See: http://docs.openstack.org/developer/"
                            "oslo.messaging/transport.html"))
            self._initial_pid = current_pid

        if retry is None:
            retry = self.max_retries
        if retry is None or retry < 0:
            retry = None

        # 失败的时候调用
        def on_error(exc, interval):
            LOG.debug("Received recoverable error from kombu:",
                      exc_info=True)
            # 有recoverable_errors,调用recoverable_errors函数
            recoverable_error_callback and recoverable_error_callback(exc)
            # 计算失败间隔
            interval = (self.kombu_reconnect_delay + interval
                        if self.kombu_reconnect_delay > 0
                        else interval)
            info = {'err_str': exc, 'sleep_time': interval}
            info.update(self.connection.info())
            if 'Socket closed' in six.text_type(exc):
                LOG.error(_LE('AMQP server %(hostname)s:%(port)s closed'
                              ' the connection. Check login credentials:'
                              ' %(err_str)s'), info)
            else:
                LOG.error(_LE('AMQP server on %(hostname)s:%(port)s is '
                              'unreachable: %(err_str)s. Trying again in '
                              '%(sleep_time)d seconds.'), info)
            # sleep到kombu_reconnect_delay<=0
            # 整个on_error主要就是用来sleep的
            if self.kombu_reconnect_delay > 0:
                LOG.trace('Delaying reconnect for %1.1f seconds ...',
                          self.kombu_reconnect_delay)
                time.sleep(self.kombu_reconnect_delay)
        # 重连调用
        def on_reconnection(new_channel):
            self.set_transport_socket_timeout()
            self._set_current_channel(new_channel)
            for consumer in self._consumers:
                # 重新声明消费者
                consumer.declare(self)
            LOG.info(_LI('Reconnected to AMQP server on '
                         '%(hostname)s:%(port)s via [%(transport)s] client'),
                     self.connection.info())
        # 执行method
        def execute_method(channel):
            self._set_current_channel(channel)
            method()
        # 我们用rabbit mq的时候
        # 这里相当于找py-amqp有没有recoverable_connection_errors
        # recoverable_errors是多个异常组成的tuple
        # 用于后面捕获异常
        # recoverable_errors是可恢复的error
        has_modern_errors = hasattr(
            self.connection.transport, 'recoverable_connection_errors',
        )
        if has_modern_errors:
            recoverable_errors = (
                self.connection.recoverable_channel_errors +
                self.connection.recoverable_connection_errors)
        else:
            recoverable_errors = ()

        try:
            # 通过kombu的autoretry生成一个自动retry的实例
            autoretry_method = self.connection.autoretry(
                execute_method, channel=self.channel,
                max_retries=retry,
                errback=on_error,
                interval_start=self.interval_start or 1,
                interval_step=self.interval_stepping,
                interval_max=self.interval_max,
                on_revive=on_reconnection)
            # 调用生成的autoretry_method并返回
            ret, channel = autoretry_method()
            self._set_current_channel(channel)
            return ret
        except recoverable_errors as exc:
            # recoverable_errors可恢复的错误
            LOG.debug("Received recoverable error from kombu:",
                      exc_info=True)
            # 触发错误回调
            error_callback and error_callback(exc)
            # 清理掉当前channel
            self._set_current_channel(None)
            # 注释说明走到这里说明重试过指定retry次数
            info = {'err_str': exc, 'retry': retry}
            info.update(self.connection.info())
            msg = _('Unable to connect to AMQP server on '
                    '%(hostname)s:%(port)s after %(retry)s '
                    'tries: %(err_str)s') % info
            LOG.error(msg)
            raise exceptions.MessageDeliveryFailure(msg)
        except rpc_amqp.AMQPDestinationNotFound:
            # 外层有对这个异常的捕获和处理
            # 一般处理是重试直至外部定义好的超时
            raise
        except Exception as exc:
            # 触发错误回调
            error_callback and error_callback(exc)
            # 抛出异常
            raise

    # 注册一个topic类型的consumer
    def declare_topic_consumer(self, exchange_name, topic, callback=None,
                               queue_name=None):
        # 生成Consumer实例
        # Consumer类中分装了kombu的Queue和Exchange
        # RabbitDriver调用declare_topic_consumer时
        # 传入的callback是AMQPListener实例
        consumer = Consumer(exchange_name=exchange_name,
                            queue_name=queue_name or topic,
                            routing_key=topic,
                            type='topic',
                            durable=self.amqp_durable_queues,
                            exchange_auto_delete=self.amqp_auto_delete,
                            queue_auto_delete=self.amqp_auto_delete,
                            callback=callback,
                            rabbit_ha_queues=self.rabbit_ha_queues)
        self.declare_consumer(consumer)

    def declare_consumer(self, consumer):

        def _connect_error(exc):
            log_info = {'topic': consumer.routing_key, 'err_str': exc}
            LOG.error(_LE("Failed to declare consumer for topic '%(topic)s': "
                          "%(err_str)s"), log_info)
        # 实际declare的函数
        def _declare_consumer():
            # 这里把self传入
            # 声明消费者,
            # amqp协议没有声明消费者,所以实际上是声明队列
            consumer.declare(self)
            tag = self._active_tags.get(consumer.queue_name)
            if tag is None:
                tag = next(self._tags)
                self._active_tags[consumer.queue_name] = tag
                self._new_tags.add(tag)
            # 每声明一个consumer
            # 就会添加到self._consumers字典中
            self._consumers[consumer] = tag
            return consumer

        with self._connection_lock:
            # 用ensure去执行_declare_consumer
            return self.ensure(_declare_consumer,
                               error_callback=_connect_error)

    def consume(self, timeout=None):
        .....
        # consume函数内容较多就不全贴了
        # 这个函数在AMQPListener的pool函数中被调用
        def _consume():
            ....
            # 有新消费者
            if self._new_tags:
                for consumer, tag in self._consumers.items():
                    if tag in self._new_tags:
                        # 先绑定消费者到队列上(同时consumer._callback会绑定到kombu的队列上)
                        consumer.consume(tag=tag)
                        self._new_tags.remove(tag)

            poll_timeout = (self._poll_timeout if timeout is None
                            else min(timeout, self._poll_timeout))
            while True:
                # 这个循环是socket超时重试用的
                if self._consume_loop_stopped:
                    return

                if self._heartbeat_supported_and_enabled():
                    self._heartbeat_check()

                try:
                    # 这里就是从获取到一帧ampq数据并调用
                    # consumer._callback然后调用AMQPListener.__call__
                    self.connection.drain_events(timeout=poll_timeout)
                    # 数据帧只收一帧
                    return
                except socket.timeout as exc:
                    poll_timeout = timer.check_return(
                        _raise_timeout, exc, maximum=self._poll_timeout)

        with self._connection_lock:
            # 用ensure函数确保执行
            self.ensure(_consume,
                        recoverable_error_callback=_recoverable_error_callback,
                        error_callback=_error_callback)


    # ----------------------下面是发送函数-----------------------------
    def direct_send(self, msg_id, msg):
        # exchange对象
        exchange = kombu.entity.Exchange(name=msg_id,
                                         type='direct',
                                         durable=False,
                                         auto_delete=True,
                                         passive=True)
        self._ensure_publishing(self._publish_and_raises_on_missing_exchange,
                                exchange, msg, routing_key=msg_id)

    def _ensure_publishing(self, method, exchange, msg, routing_key=None,
                           timeout=None, retry=None):
        def _error_callback(exc):
            log_info = {'topic': exchange.name, 'err_str': exc}
            LOG.error(_LE("Failed to publish message to topic "
                          "'%(topic)s': %(err_str)s"), log_info)
            LOG.debug('Exception', exc_info=exc)
        # 因为ensure的method不能带参数
        # 所以这里用functools.partial把参数绑定了
        # method是绑定了参数的_publish_and_raises_on_missing_exchange
        method = functools.partial(method, exchange, msg, routing_key, timeout)
        with self._connection_lock:
            self.ensure(method, retry=retry, error_callback=_error_callback)

    def _publish_and_raises_on_missing_exchange(self, exchange, msg,
                                                routing_key=None,
                                                timeout=None):
        # exchange的passive属性必须是true
        if not exchange.passive:
            raise RuntimeError("_publish_and_retry_on_missing_exchange() must "
                               "be called with an passive exchange.")
        try:
            # 调用下面的_publish推送消息
            self._publish(exchange, msg, routing_key=routing_key,
                          timeout=timeout)
            return
        except self.connection.channel_errors as exc:
            if exc.code == 404:
                # 404是AMQPDestinationNotFound错误
                raise rpc_amqp.AMQPDestinationNotFound(
                    "exchange %s doesn't exists" % exchange.name)
            raise

    def _publish(self, exchange, msg, routing_key=None, timeout=None):
        # 创建一个kombu的producer对象
        producer = kombu.messaging.Producer(exchange=exchange,
                                            channel=self.channel,
                                            auto_declare=not exchange.passive,
                                            routing_key=routing_key)

        log_info = {'msg': msg,
                    'who': exchange or 'default',
                    'key': routing_key}
        LOG.trace('Connection._publish: sending message %(msg)s to'
                  ' %(who)s with routing key %(key)s', log_info)
        # 在socket超时时间内publish数据
        with self._transport_socket_timeout(timeout):
            producer.publish(msg, expiration=self._get_expiration(timeout),
                             compression=self.kombu_compression)
    .....
    # 省略其他代码

# Consumer类
class Consumer(object):
    def __init__(self, exchange_name, queue_name, routing_key, type, durable,
                 exchange_auto_delete, queue_auto_delete, callback,
                 nowait=True, rabbit_ha_queues=None, rabbit_queue_ttl=0):
        # Consumer的属性大部分是amqp相关的参数
        self.queue_name = queue_name
        self.exchange_name = exchange_name
        self.routing_key = routing_key
        self.exchange_auto_delete = exchange_auto_delete
        self.queue_auto_delete = queue_auto_delete
        self.durable = durable
        # RabbitDriver中declare_topic_consumer时
        # 传入的callback是AMQPListener实例
        self.callback = callback
        self.type = type
        self.nowait = nowait
        # 这里是把两个配置文件组成字典
        self.queue_arguments = _get_queue_arguments(rabbit_ha_queues,
                                                    rabbit_queue_ttl)

        self.queue = None
        self.exchange = kombu.entity.Exchange(
            name=exchange_name,
            type=type,
            durable=self.durable,
            auto_delete=self.exchange_auto_delete)

    def declare(self, conn):
        # 声明队列
        # 先生成kombu的Queue实例
        self.queue = kombu.entity.Queue(
            name=self.queue_name,
            channel=conn.channel,
            exchange=self.exchange,
            durable=self.durable,
            auto_delete=self.queue_auto_delete,
            routing_key=self.routing_key,
            queue_arguments=self.queue_arguments)

        try:
            LOG.trace('ConsumerBase.declare: '
                      'queue %s', self.queue_name)
            # 声明当前queue
            self.queue.declare()
        except conn.connection.channel_errors as exc:
            # NOTE(jrosenboom): This exception may be triggered by a race
            # condition. Simply retrying will solve the error most of the time
            # and should work well enough as a workaround until the race
            # condition itself can be fixed.
            # See https://bugs.launchpad.net/neutron/+bug/1318721 for details.
            if exc.code == 404:
                self.queue.declare()
            else:
                raise

    def consume(self, tag):
        # 当前消费者订阅队列
        # 订阅的时候把self._callback传入kombu.entity.Queue中
        # connection调用consume的时候
        # 这里才被调用,也就是这时候才开始订阅
        # 参考前面Connection类的consume方法说明
        self.queue.consume(callback=self._callback,
                           consumer_tag=six.text_type(tag),
                           nowait=self.nowait)

    def cancel(self, tag):
        # 取消队列订阅
        LOG.trace('ConsumerBase.cancel: canceling %s', tag)
        self.queue.cancel(six.text_type(tag))

    def _callback(self, message):
        # 回调函数
        # 当前函数是在consume函数订阅队列时候作为callback
        # 传入kombu.entity.Queue中,所以message的类型需要到kombu中看
        # ---------------------------------#
        # channel中有message_to_python方法就调用message_to_python
        # 转换传入的message
        m2p = getattr(self.queue.channel, 'message_to_python', None)
        if m2p:
            message = m2p(message)
        # 调用回调
        try:
            # 也就是说,AMQPListener的__call__的参数是
            # 传入的是RabbitMessage的实例
            # 也就是说AMQPListener中
            # AMQPIncomingMessage封装的message
            # 是RabbitMessage
            self.callback(RabbitMessage(message))
        except Exception:
            LOG.exception(_LE("Failed to process message"
                              " ... skipping it."))
            # 转化为RabbitMessage或者调用callback报错
            # 回复ack
            message.ack()

class RabbitMessage(dict):
    # RabbitMessage是个字典,简单的封装了一下原始message
    # 主要功能作为一个普通字典, 数据来源raw_message.payload['oslo.message']
    # 多了两个方法
    # acknowledge和requeue
    def __init__(self, raw_message):
        super(RabbitMessage, self).__init__(
            # deserialize_msg主要顺便检查了消息的版本号
            # 然后取出raw_message.payload['oslo.message']部分数据
            # 并转化这部分数据为dict
            rpc_common.deserialize_msg(raw_message.payload))
        LOG.trace('RabbitMessage.Init: message %s', self)
        self._raw_message = raw_message

    def acknowledge(self):
        LOG.trace('RabbitMessage.acknowledge: message %s', self)
        self._raw_message.ack()

    def requeue(self):
        LOG.trace('RabbitMessage.requeue: message %s', self)
        self._raw_message.requeue()

```


我们来看看序列化和反序列化

```python

_VERSION_KEY = 'oslo.version'
_MESSAGE_KEY = 'oslo.message'

def deserialize_msg(msg):
    # 外部调用的是rpc_common.deserialize_msg(raw_message.payload))
    # 这里也就是说raw_message.payload一定是字典
    if not isinstance(msg, dict):
        return msg
    base_envelope_keys = (_VERSION_KEY, _MESSAGE_KEY)
    # 这个写法是确保msg的字典包含有base_envelope_keys
    # 相当于双循环找key,这个写法比较漂亮
    if not all(map(lambda key: key in msg, base_envelope_keys)):
        return msg
    # 从oslo.version中取出版本号,确认版本号兼容
    if not utils.version_is_compatible(_RPC_ENVELOPE_VERSION,
                                       msg[_VERSION_KEY]):
        raise UnsupportedRpcEnvelopeVersion(version=msg[_VERSION_KEY])
    # 从oslo.message中取出数据转成dict并返回
    raw_msg = jsonutils.loads(msg[_MESSAGE_KEY])
    return raw_msg

# AMQPListener从message里获取context_dict的方法也看一下
# 这里传入的msg也就是RabbitMessage
def unpack_context(msg):
    context_dict = {}
    # 从RabbitMessage中取出所有_context_打头的key和value
    for key in list(msg.keys()):
        key = six.text_type(key)
        if key.startswith('_context_'):
            value = msg.pop(key)
            context_dict[key[9:]] = value
    # 取出key是_msg_id和_reply_q的key
    context_dict['msg_id'] = msg.pop('_msg_id', None)
    context_dict['reply_q'] = msg.pop('_reply_q', None)
    # 转成RpcContext实例
    # 也就是说unpack_context返回的不是dict而是
    # RpcContext实例
    # 主要内容上面也比较明了
    # 取出部分数据, 也就是把RabbitMessage的数据再拆分走一部分
    return RpcContext.from_dict(context_dict)


class RpcContext(rpc_common.CommonRpcContext):
    """Context that supports replying to a rpc.call."""
    def __init__(self, **kwargs):
        self.msg_id = kwargs.pop('msg_id', None)
        self.reply_q = kwargs.pop('reply_q', None)
        super(RpcContext, self).__init__(**kwargs)

    def deepcopy(self):
        values = self.to_dict()
        values['conf'] = self.conf
        values['msg_id'] = self.msg_id
        values['reply_q'] = self.reply_q
        return self.__class__(**values)

class CommonRpcContext(object):
    def __init__(self, **kwargs):
        # self.values就是前面的context_dict字典
        self.values = kwargs

    def __getattr__(self, key):
        # 模拟字典
        try:
            return self.values[key]
        except KeyError:
            raise AttributeError(key)

    def to_dict(self):
        # 可以看到to_dict方法是个
        # self.values就是前面的context_dict字典
        return copy.deepcopy(self.values)

    @classmethod
    def from_dict(cls, values):
        return cls(**values)

    def deepcopy(self):
        return self.from_dict(self.to_dict())

    def update_store(self):
        # local.store.context = self
        pass

```

接下来要完全搞清楚原生message是什么,需要看kombu和py-amqp
