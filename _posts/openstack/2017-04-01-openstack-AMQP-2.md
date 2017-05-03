---
layout: post
title:  "OpenStack Mitaka从零开始 openstack里的AMQP使用(2)"
date:   2017-04-01 12:50:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}



我们现在来看看RPCDispatcher,还是和上次一样,我们的服务原型是l3-agent

```python
# 因为要用到transport._listen,这里也贴一下
class Transport(object):
    .....
    def _listen(self, target):
        # 调用_listen, 必须同时有topic、和server属性
        if not (target.topic and target.server):
         raise exceptions.InvalidTarget('A server\'s target must have '
                                        'topic and server names specified',
                                        target)
        # 这里初始化rabbit驱动的监听
        return self._driver.listen(target)
    ...


def get_rpc_server(transport, target, endpoints,
                executor='blocking', serializer=None):
     # dispatcher就是分发器,RPCDispatcher就是这一节的重点
     dispatcher = rpc_dispatcher.RPCDispatcher(target, endpoints, serializer)
     # MessageHandlingServer就是rpc server
     return msg_server.MessageHandlingServer(transport, dispatcher, executor)



# RPCDispatcher的代码
class RPCDispatcher(dispatcher.DispatcherBase):
    """A message dispatcher which understands RPC messages.

    A MessageHandlingServer is constructed by passing a callable dispatcher
    which is invoked with context and message dictionaries each time a message
    is received.

    RPCDispatcher is one such dispatcher which understands the format of RPC
    messages. The dispatcher looks at the namespace, version and method values
    in the message and matches those against a list of available endpoints.

    Endpoints may have a target attribute describing the namespace and version
    of the methods exposed by that object. All public methods on an endpoint
    object are remotely invokable by clients.
    能理解RPC信息的消息分发器
    MessageHandlingServer通过一个callable的dispatcher来构建的的
    每当有消息过来的时候dispatcher就被调用,调用的他的参数中包含的数据包含有context和消息字典

    RPCDispatcher就是一个能理解RPC消息的分发器
    分发器监听消息中的namespace, version and method值
    并从列表中匹配到对应的endpoint,后面就不翻译了

    """
    # DispatcherBase主要有以下两个属性
    # 这值设置分发器一次只处理一个消息
    # 这个属性在RPC server(MessageHandlingServer)的循环中有使用
    # 可以看到是固定值1
    batch_size = 1
    batch_timeout = None

    def __init__(self, target, endpoints, serializer):
        self.endpoints = endpoints
        self.serializer = serializer or msg_serializer.NoOpSerializer()
        self._default_target = msg_target.Target()
        self._target = target

    # listen实际是transport._listen
    def _listen(self, transport):
        return transport._listen(self._target)

    # 后面有说明
    @staticmethod
    def _is_namespace(target, namespace):
        return namespace in target.accepted_namespaces

    # 后面有说明
    @staticmethod
    def _is_compatible(target, version):
        endpoint_version = target.version or '1.0'
        return utils.version_is_compatible(endpoint_version, version)

    # Rpc server,也就是MessageHandlingServer,调用RPCDispatcher的方式
    # 调用位置在下一节有说明
    def __call__(self, incoming):
        # 这个acknowledge肯定是回复rabbitmq ack
        incoming[0].acknowledge()
        # 调用call返回的是DispatcherExecutorContext实例
        # 外部最后会调用DispatcherExecutorContext.run方法
        # 这里其实是在DispatcherExecutorContext里绕了一下
        # DispatcherExecutorContext.run最后也是
        # 调用RPCDispatcher的_dispatch_and_reply去处理消息的
        # 重点在于incoming[0]是什么,_dispatch_and_reply里又做了什么
        # incoming[0]是什么后面会在后面一节说明
        return dispatcher.DispatcherExecutorContext(
            incoming[0], self._dispatch_and_reply)

    def _dispatch_and_reply(self, incoming):
        # 从名字可以看出, 用于分发和应答
        # incoming就是前面的incoming[0]
        try:
            # 调用incoming.reply
            # incoming.reply用于应答rpc调用
            # 也就是返回self._dispatch(incoming.ctxt,incoming.message)
            # 执行后的的返回值
            # 实际工作还是在self._dispatch里
            # 用于最终分发rpc消息具体落地操作是在self._dispatch中
            # 至于incoming.reply如何应答消息的后面相关章节有说明
            incoming.reply(self._dispatch(incoming.ctxt,
                                          incoming.message))
        # 后面是捕获Exception后的处理
        # 可以看到捕获期待的错误后也会通过incoming.reply回复
        except ExpectedException as e:
            LOG.debug(u'Expected exception during message handling (%s)',
                      e.exc_info[1])
            incoming.reply(failure=e.exc_info, log_failure=False)
        except Exception as e:
            # current sys.exc_info() content can be overriden
            # by another exception raise by a log handler during
            # LOG.exception(). So keep a copy and delete it later.
            exc_info = sys.exc_info()
            try:
                LOG.error(_LE('Exception during message handling: %s'), e,
                          exc_info=exc_info)
                incoming.reply(failure=exc_info)
            finally:
                # NOTE(dhellmann): Remove circular object reference
                # between the current stack frame and the traceback in
                # exc_info.
                del exc_info

    def _dispatch(self, ctxt, message):
        # 这个就是对消息具体分发的过程
        # method就是具体的方法名
        method = message.get('method')
        # 参数
        args = message.get('args', {})
        # endpoint的命名空间,一般是None
        namespace = message.get('namespace')
        # 版本号,和target一样默认是1.0
        version = message.get('version', '1.0')
        found_compatible = False
        for endpoint in self.endpoints:
            target = getattr(endpoint, 'target', None)
            if not target:
                target = self._default_target
            # 判断消息在不在target的namespace中
            # l3-agent的namespace列表里只有一个None
            # 收到的namespace也应该是None或者没有设置namespace
            if not (self._is_namespace(target, namespace) and
                   # 判断target的版本是否和发送来的版本兼容
                   # 如果target没有设置version
                   # target默认版本号默认为1.0
                    self._is_compatible(target, version)):
                continue
            # 在endpoint中找到对应method
            if hasattr(endpoint, method):
                localcontext._set_local_context(ctxt)
                try:
                    # 调用_do_dispatch
                    # 这里又封装了一次
                    # _dispatch是为了找到有对应method的endpoint
                    # _do_dispatch是调用endpoint中的对应方法处理信息
                    # 返回值是endpoint的落地操作的结果
                    # 返回值作为incoming.reply的参数
                    # 也就是让incoming.reply函数去返回返回值
                    return self._do_dispatch(endpoint, method, ctxt, args)
                finally:
                    localcontext._clear_local_context()

            found_compatible = True

        # 没有在前面返回,抛出异常
        if found_compatible:
            raise NoSuchMethod(method)
        else:
            raise UnsupportedVersion(version, method=method)

    def _do_dispatch(self, endpoint, method, ctxt, args):
        ctxt = self.serializer.deserialize_context(ctxt)
        new_args = dict()
        for argname, arg in six.iteritems(args):
            new_args[argname] = self.serializer.deserialize_entity(ctxt, arg)
        # 通过method在endpoint中找到对应的方法
        # 再强调一次,这里的endpoint是 neutron.agent.l3.agent.L3NATAgent
        # 这里就是具体的落地操作了
        func = getattr(endpoint, method)
        result = func(ctxt, **new_args)
        return self.serializer.serialize_entity(ctxt, result)

class DispatcherExecutorContext(object):
    def __init__(self, incoming, dispatch, post=None):
        self._result = None
        self._incoming = incoming
        self._dispatch = dispatch
        self._post = post

    def run(self):
        # 可以看到run是调用dispatch的入口
        # 所以外部需要调用下DispatcherExecutorContext的run的方法才开始执行具体操作
        # 调用位置在下一节有说明
        try:
            self._result = self._dispatch(self._incoming)
        except Exception:
            msg = _('The dispatcher method must catches all exceptions')
            LOG.exception(msg)
            raise RuntimeError(msg)

    def done(self):
        # 完成通知函数
        # 把接受到的消息，和处理传给post函数
        # RPCDispatcher里done是none
        # _dispatch也是没有返回值的
        # 这个done在RPCDispatcher中没用
        # 这里可以看出有done实例需要self._dispatch有返回值
        if self._post is not None:
            self._post(self._incoming, self._result)

```

现在剩下的问题是,MessageHandlingServe和incoming的结构和,请看[下一节](http://www.lolizeppelin.com/2017/04/01/openstack-AMQP-3/)
