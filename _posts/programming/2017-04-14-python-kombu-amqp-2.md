---
layout: post
title:  "python kombu与amqp(1)"
date:   2017-04-14 12:50:00 +0800
categories: "编程"
tag: ["python", "linux"]
---

* content
{:toc}



来看看Transport


```python
 # kombu.transport.pyamqp.Transport

 class Transport(base.Transport):

     Connection = Connection
     ...

     # client是kombu.connection.Connection
     def __init__(self, client,
                  default_port=None, default_ssl_port=None, **kwargs):
         self.client = client
         self.default_port = default_port or self.default_port
         self.default_ssl_port = default_ssl_port or self.default_ssl_port
         ....

     def establish_connection(self):
         """Establish connection to the AMQP broker."""
         conninfo = self.client
         for name, default_value in items(self.default_connection_params):
             if not getattr(conninfo, name, None):
                 setattr(conninfo, name, default_value)
         if conninfo.hostname == 'localhost':
             conninfo.hostname = '127.0.0.1'
         opts = dict({
             'host': conninfo.host,
             'userid': conninfo.userid,
             'password': conninfo.password,
             'login_method': conninfo.login_method,
             'virtual_host': conninfo.virtual_host,
             'insist': conninfo.insist,
             'ssl': conninfo.ssl,
             'connect_timeout': conninfo.connect_timeout,
             'heartbeat': conninfo.heartbeat,
         }, **conninfo.transport_options or {})
         conn = self.Connection(**opts)
         conn.client = self.client
         conn.connect()
         return conn

# amqp.connection.Connection
class Connection(AbstractChannel):

    Channel = Channel

    def __init__(self, ....):
        self.frame_handler_cls = frame_handler
        # init里主要是参数初始化

    # 上下文方式
    def __enter__(self):
        self.connect()
        return self

    def __exit__(self, *eargs):
        self.close()

    # 注入各种回调
    def _setup_listeners(self):
        self._callbacks.update({
            spec.Connection.Start: self._on_start,
            spec.Connection.OpenOk: self._on_open_ok,
            spec.Connection.Secure: self._on_secure,
            spec.Connection.Tune: self._on_tune,
            spec.Connection.Close: self._on_close,
            spec.Connection.Blocked: self._on_blocked,
            spec.Connection.Unblocked: self._on_unblocked,
            spec.Connection.CloseOk: self._on_close_ok,
        })

    def _on_start(self, version_major, version_minor, server_properties,
                  mechanisms, locales, argsig='FsSs'):
        .....
        # 省略部分代码
        # _on_start回调是发送一个StartOk标记
        self.send_method(
            spec.Connection.StartOk, argsig,
            (client_properties, authentication.mechanism,
             login_response, self.locale),
        )

    def _on_open_ok(self):
        #_on_open_ok回调设置_handshake_complete
        self._handshake_complete = True
        # on_open又是个兼容python 2、3的处理
        self.on_open(self)

    def connect(self, callback=None):
        # socket链接在这
        if self.connected:
            # 再已经链接的情况调用connect的
            # 直接执行callback然后返回
            return callback() if callback else None
        # 具体的网络层
        self.transport = self.Transport(
            self.host, self.connect_timeout, self.ssl,
            self.read_timeout, self.write_timeout,
            socket_settings=self.socket_settings,
        )
        # 调用网络层链接
        # transport是amqp.transport.TCPTransport
        self.transport.connect()
        # 初始化on_inbound_frame属性
        # 应该是帧到来处理类
        # on_inbound_frame是一个闭包
        # 下面有具体说明on_inbound_frame的功能
        self.on_inbound_frame = self.frame_handler_cls(
            self, self.on_inbound_method)
        # 帧写入类
        self.frame_writer = self.frame_writer_cls(self, self.transport)
        # 在握手完毕之前,也就是收到OpenOk回复
        while not self._handshake_complete:
            self.drain_events(timeout=self.connect_timeout)

    # 上面on_inbound_frame初始化的是后传入的方法
    def on_inbound_method(self, channel_id, method_sig, payload, content):
        # 调用channel的dispatch_method方法分发函数
        return self.channels[channel_id].dispatch_method(
            method_sig, payload, content,
        )

    # 接受数据要循环调用drain_events
    # 也就是要用死循环调用drain_events来收数据
    def drain_events(self, timeout=None):
        return self.blocking_read(timeout)

    def blocking_read(self, timeout=None):
        with self.transport.having_timeout(timeout):
            # 从socket里读取到完整的帧
            frame = self.transport.read_frame()
        # 接收到的帧调用on_inbound_frame去处理
        return self.on_inbound_frame(frame)

```

我我们来看看on_inbound_frame属性的内容

```python
def frame_handler(connection, callback,
                  unpack_from=unpack_from, content_methods=_CONTENT_METHODS):
    # 这个就是amqp.connection.Connection的on_inbound_frame属性
    # 第一个参数就是amqp.connection.Connection自身
    # callback就是amqp.connection.Connection.on_inbound_method
    # 也就是数据分发是通过on_inbound_method中的
    # self.channels[channel_id].dispatch_method去处理
    expected_types = defaultdict(lambda: 1)
    partial_messages = {}

    def on_frame(frame):
        # 帧数据分为  帧数类型  channel 和 buf
        # 下面都是根据帧类型分发处理帧
        frame_type, channel, buf = frame
        connection.bytes_recv += 1
        if frame_type not in (expected_types[channel], 8):
            raise UnexpectedFrame(
                'Received frame {0} while expecting type: {1}'.format(
                    frame_type, expected_types[channel]),
            )
        elif frame_type == 1:
            method_sig = unpack_from('>HH', buf, 0)

            if method_sig in content_methods:
                # Save what we've got so far and wait for the content-header
                partial_messages[channel] = Message(
                    frame_method=method_sig, frame_args=buf,
                )
                expected_types[channel] = 2
            else:
                callback(channel, method_sig, buf, None)

        elif frame_type == 2:
            msg = partial_messages[channel]
            msg.inbound_header(buf)
            if msg.ready:
                # bodyless message, we're done
                expected_types[channel] = 1
                partial_messages.pop(channel, None)
                callback(channel, msg.frame_method, msg.frame_args, msg)
            else:
                # wait for the content-body
                expected_types[channel] = 3
        elif frame_type == 3:
            msg = partial_messages[channel]
            msg.inbound_body(buf)
            if msg.ready:
                expected_types[channel] = 1
                partial_messages.pop(channel, None)
                callback(channel, msg.frame_method, msg.frame_args, msg)
        elif frame_type == 8:
            # bytes_recv already updated
            pass
    return on_frame
```
