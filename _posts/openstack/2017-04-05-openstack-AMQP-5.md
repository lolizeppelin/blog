---
layout: post
title:  "OpenStack Mitaka从零开始 openstack里的AMQP使用(5)"
date:   2017-04-05 12:50:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}


```python


class Connection(object):
    """Connection object."""

    pools = {}

    def __init__(self, conf, url, purpose):
        # NOTE(viktors): Parse config options
        driver_conf = conf.oslo_messaging_rabbit
        ......
        # 省略属性部分
        # Try forever?
        if self.max_retries <= 0:
            self.max_retries = None

        if url.virtual_host is not None:
            virtual_host = url.virtual_host
        else:
            virtual_host = self.virtual_host

        self._url = ''
        if self.fake_rabbit:
            # 假冒rabbit
            ....
        elif url.hosts:
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
        self.channel = None
        self.purpose = purpose

        # send和listen的锁不一样
        if purpose == rpc_common.PURPOSE_SEND:
            self._connection_lock = ConnectionLock()
        else:
            self._connection_lock = DummyConnectionLock()
        # 初始化kombu的Connection
        # socket数据的收发,amqp消息的封装生成都在kombu里完成
        # kombu最终是调用py-ampq封装成amqp数据包的
        self.connection = kombu.connection.Connection(
            self._url, ssl=self._fetch_ssl_params(),
            # login_method没什么大用,不同登陆模式区别看
            # https://www.rabbitmq.com/authentication.html
            login_method=self.login_method,
            heartbeat=self.heartbeat_timeout_threshold,
            failover_strategy=self.kombu_failover_strategy,
            # 这个是py-ampq用的参数
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
        # 如果kombu的connection不支持心跳
        # 设置_heartbeat_support_log_emitted参数为true后再返回False
        # _heartbeat_support_log_emitted我没发现有什么用
        elif not self._heartbeat_support_log_emitted:
            LOG.warning(_LW("Heartbeat support requested but it is not "
                            "supported by the kombu driver or the broker"))
            self._heartbeat_support_log_emitted = True
        return False

    def ensure_connection(self):
        self._set_current_channel(None)
        # ensure比较复杂,method需要callable
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
                # 重新declare
                consumer.declare(self)
            LOG.info(_LI('Reconnected to AMQP server on '
                         '%(hostname)s:%(port)s via [%(transport)s] client'),
                     self.connection.info())
        # 执行method
        def execute_method(channel):
            self._set_current_channel(channel)
            method()
        # 我们用rabbit mq的时候
        # 这里相当于找py-ampq有没有recoverable_connection_errors
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

    # 注册一个direct类型的consumer,也就是direct的queue
    def declare_direct_consumer(self, topic, callback):
        # 生成Consumer实例
        consumer = Consumer(exchange_name=topic,
                            queue_name=topic,
                            routing_key=topic,
                            type='direct',
                            durable=False,
                            exchange_auto_delete=True,
                            queue_auto_delete=False,
                            callback=callback,
                            rabbit_ha_queues=self.rabbit_ha_queues,
                            rabbit_queue_ttl=self.rabbit_transient_queues_ttl)
        self.declare_consumer(consumer)

    def declare_consumer(self, consumer):

        def _connect_error(exc):
            log_info = {'topic': consumer.routing_key, 'err_str': exc}
            LOG.error(_LE("Failed to declare consumer for topic '%(topic)s': "
                          "%(err_str)s"), log_info)
        # 实际declare的函数
        def _declare_consumer():
            # 这里把self传入
            consumer.declare(self)
            tag = self._active_tags.get(consumer.queue_name)
            if tag is None:
                tag = next(self._tags)
                self._active_tags[consumer.queue_name] = tag
                self._new_tags.add(tag)
            self._consumers[consumer] = tag
            return consumer

        with self._connection_lock:
            # 用ensure去执行_declare_consumer
            return self.ensure(_declare_consumer,
                               error_callback=_connect_error)

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


```
