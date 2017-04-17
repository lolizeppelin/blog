---
layout: post
title:  "python kombu与amqp(2)"
date:   2017-04-14 12:50:00 +0800
categories: "编程"
tag: ["python", "linux"]
---

* content
{:toc}


来看看kombu.transport.pyamqp.Transport

```python
 # kombu.transport.pyamqp.Transport

 class Transport(base.Transport):
     # Connection类
     # amqp.connection.Connection
     Connection = Connection
     ...

     # client是kombu.connection.Connection
     def __init__(self, client,
                  default_port=None, default_ssl_port=None, **kwargs):
         # client是kombu.connection.Connection类实例
         # 注意和这里的Connection不是一个
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
         # 生成Connection类实例
         conn = self.Connection(**opts)
         # client也传入
         conn.client = self.client
         conn.connect()
         return conn

# amqp.connection.Connection
class Connection(AbstractChannel):

    Channel = Channel

    def __init__(self, ....):
        self.frame_handler_cls = frame_handler
        # init里主要是参数初始化

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
    # 也就是外部要用死循环调用drain_events来收数据
    def drain_events(self, timeout=None):
        return self.blocking_read(timeout)

    def blocking_read(self, timeout=None):
        with self.transport.having_timeout(timeout):
            # 从socket里读取到完整的帧, 帧结构要看read_frame
            frame = self.transport.read_frame()
        # 接收到的帧调用on_inbound_frame去处理
        return self.on_inbound_frame(frame)
```

我我们来看看on_inbound_frame属性的内容

```python
# 是下面默认的content_methods
# frozenset是不可变的set,可以直接当字典的key
_CONTENT_METHODS = frozenset([
    spec.Basic.Return,
    spec.Basic.Deliver,
    spec.Basic.GetOk,
])

def frame_handler(connection, callback,
                  unpack_from=unpack_from, content_methods=_CONTENT_METHODS):
    # 这个就是amqp.connection.Connection的on_inbound_frame属性
    # 第一个参数就是amqp.connection.Connection自身
    # callback就是amqp.connection.Connection.on_inbound_method
    # 也就是数据分发是通过on_inbound_method中的
    # self.channels[channel_id].dispatch_method去处理
    # unpack_from默认就是struct.unpack_from
    # ------------------------------------------------------#
    # defaultdict用法是带有默认值的dict,例子参考下面可以做单词统计的字典
    # counts = defaultdict(lambda: 0)
    # for s in strings:
    #    counts[s] += 1
    # expected_types的默认值保证任意channel都有一个1的返回值
    # expected_types是一个自由变量
    # 用于存放每个channel当前期待的frame_type值
    expected_types = defaultdict(lambda: 1)
    # 用于缓存帧数据的字典,key也是channel
    # 也是自由变量
    partial_messages = {}
    def on_frame(frame):
        # 可以看出帧数据分为  帧数类型  channel 和 buf
        # 帧结构要看read_frame
        # 下面都是根据帧类型分发处理帧
        frame_type, channel, buf = frame
        # 链接收帧数+1
        connection.bytes_recv += 1
        # 后面会不停的修改expected_types[channel]的值
        # 来保证每个channel下次收到的帧的的type正确
        if frame_type not in (expected_types[channel], 8):
            # 帧type不是期待的type
            raise UnexpectedFrame(
                'Received frame {0} while expecting type: {1}'.format(
                    frame_type, expected_types[channel]),
            )
        elif frame_type == 1:
            # TYPE 1 是第一个包
            method_sig = unpack_from('>HH', buf, 0)
            if method_sig in content_methods:
                # 放进缓存字典,Message类在下面看
                partial_messages[channel] = Message(
                    frame_method=method_sig, frame_args=buf,
                )
                # 修改channel对应的type期待值的值为2
                expected_types[channel] = 2
            else:
                # callback就是amqp.connection.Connection.on_inbound_method
                # 对应参数channel_id, method_sig, payload, content
                callback(channel, method_sig, buf, None)
        elif frame_type == 2:
            # TYPE 2是设置帧属性
            # 从缓存字典里取出
            msg = partial_messages[channel]
            # 插入头部信息
            msg.inbound_header(buf)
            # 当前帧已完整
            if msg.ready:
                # 还原期待type为1
                expected_types[channel] = 1
                # 从缓存字典中弹出
                partial_messages.pop(channel, None)
                callback(channel, msg.frame_method, msg.frame_args, msg)
            # 当前帧不完整
            else:
                # 收包体
                expected_types[channel] = 3
        elif frame_type == 3:
            # TYPE 3 是添加包体
            msg = partial_messages[channel]
            # 插入包体信息
            msg.inbound_body(buf)
            # 帧已完整
            if msg.ready:
                expected_types[channel] = 1
                partial_messages.pop(channel, None)
                callback(channel, msg.frame_method, msg.frame_args, msg)
        elif frame_type == 8:
            # bytes_recv already updated
            pass
    return on_frame
    # 帧数据合并不能跨connection,所以接收数据的connection只能有一个
```

现在我们来看看on_frame中的Message类


```python

# 初始化
# partial_messages[channel] = Message(
#     frame_method=method_sig, frame_args=buf,
# )
# 插入头部信息
# msg.inbound_header(buf)
# 插入包体信息
# msg.inbound_body(buf)

class Message(GenericContent):
    CLASS_ID = Basic.CLASS_ID

    #: Instances of this class have these attributes, which
    #: are passed back and forth as message properties between
    #: client and server
    PROPERTIES = [
        ('content_type', 's'),
        ('content_encoding', 's'),
        ('application_headers', 'F'),
        ('delivery_mode', 'o'),
        ('priority', 'o'),
        ('correlation_id', 's'),
        ('reply_to', 's'),
        ('expiration', 's'),
        ('message_id', 's'),
        ('timestamp', 'L'),
        ('type', 's'),
        ('user_id', 's'),
        ('app_id', 's'),
        ('cluster_id', 's')
    ]

    #: set by basic_consume/basic_get
    delivery_info = None

    def __init__(self, body='', children=None, channel=None, **properties):
        super(Message, self).__init__(**properties)
        self.body = body
        self.channel = channel

    @property
    def headers(self):
        return self.properties.get('application_headers')

    @property
    def delivery_tag(self):
        return self.delivery_info.get('delivery_tag')

# 对应类型的解包函数
PROPERTY_CLASSES = {
    # Basic.CLASS_ID的解包函数decode_properties_basic
    Basic.CLASS_ID: decode_properties_basic,
}

class GenericContent(object):
    CLASS_ID = None
    PROPERTIES = [('dummy', 's')]

    def __init__(self, frame_method=None, frame_args=None, **props):
        # 前面收到type 为1的包的
        # method_sig是frame_method
        # buf是frame_args
        # 要知道frame_method和frame_args
        # 要看self.transport.read_frame()返回的type是1的帧
        self.frame_method = frame_method
        self.frame_args = frame_args

        self.properties = props
        self._pending_chunks = []
        self.body_received = 0
        self.body_size = 0
        self.ready = False

    def __getattr__(self, name):
        # Look for additional properties in the 'properties'
        # dictionary, and if present - the 'delivery_info' dictionary.
        if name == '__setstate__':
            # Allows pickling/unpickling to work
            raise AttributeError('__setstate__')

        if name in self.properties:
            return self.properties[name]
        raise AttributeError(name)

    def _load_properties(self, class_id, buf, offset=0,
                         classes=PROPERTY_CLASSES, unpack_from=unpack_from):
        # Message类的CLASS_ID是Basic.CLASS_ID
        # classes[class_id]就是decode_properties_basic函数
        # decode_properties_basic是一根据buf, offset生成props并返回新的offset
        props, offset = classes[class_id](buf, offset)
        # inbound_header其实就是设置属性
        self.properties = props
        return offset

    def _serialize_properties(self):
        .....
        # 这个函数是write的时候才被调用序列化数据的
        # 我们看接收部分可以掠过

    def inbound_header(self, buf, offset=0):
        # 因为基本都是处理整个buf,buf分包粘包都处理过的
        # 所以offset一般都是0
        # class_id肯定是Basic.CLASS_ID
        # 在这里设置了当前msg的包体长度
        class_id, self.body_size = unpack_from('>HxxQ', buf, offset)
        # 偏移12,剔除 class_id 2 body_size 8 两个空 2 一共12
        # len(struct.pack('>HxxQ', 1, 1)) == 12
        offset += 12
        self._load_properties(class_id, buf, offset)
        # 没有包体
        if not self.body_size:
            self.ready = True
        return offset

    def inbound_body(self, buf):
        # 缓存包体的列表
        chunks = self._pending_chunks
        self.body_received += len(buf)
        # 已经到整包大小
        if self.body_received >= self.body_size:
            # 缓存列表有数据
            # 用了列表不直接字符串相加
            # 说明包体有可能很大?
            if chunks:
                chunks.append(buf)
                # 合并包体?没有切片剔除超过body_size的部分
                # 兼容python2和3所有没用''.join
                self.body = bytes().join(chunks)
                # 清空缓存
                # 写del chunks[:]比较好看
                chunks[:] = []
            else:
                self.body = buf
            self.ready = True
            # 从上面处理可以看出
            # 1、最后实际的body大小有可能大于body_size
            # 2、没有判断ready,收完包后再调用inbound_body会覆盖掉body
            # 3、inbound_header也有类似问题
        else:
            chunks.append(buf)
```

现在剩下2个问题

1. transport.read_frame()生成返回值的过程
2. channel的dispatch_method的过程(如何调用外城传入的callback)
