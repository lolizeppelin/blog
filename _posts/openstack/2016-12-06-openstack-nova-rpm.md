---
layout: post
title:  "OpenStack Mitaka从零开始 通过nova的rpm包结构分析nova组件功能 k"
date:   2016-12-06 15:05:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}


nova的每一个rpm包很大程度上表示了每一个具体的功能

## nova的rpm包拆分部分成如下部分

一、python-nova

    核心代码全部在python-nova中,nova的所有功能代码都在里面
    从这里可以看出,openstack的代码都还没拆分好,所以所有代码都丢一块了
    装任意一个nova的一个组件,都要把几乎所有nova的代码带上


二、python-nova-tests

    测试用


三、openstack-nova

    只有一个nova-all脚本  貌似是所有nova console的入口

    四、openstack-nova-api
    rpm包里有3服务启动文件和对应的systemd配置文件，3个服务分别是
    1、nova-api-metadata   代码位置nova.cmd.api_metadata
    参考http://blog.chinaunix.net/uid-7374279-id-4985397.html
    参考摘要：metadata字面上是元数据，是一个不容易理解的概念。在除了openstack的其他场合也经常会碰到。openstack里的metadata，是提供一个机制给用户，可以设定每一个instance 的参数。
    比如你想给instance设置某个属性，比如主机名。metadata的一个重要应用，是设置每个instance的ssh公钥。公钥的设置有两种方式：
     1、创建instance时注入文件镜像
     2、启动instance后，通过metadata获取，然后用脚本写入
    总结：用于存放虚拟机的特殊数据(静态数据、例如主机名字、ssh公钥之类)的服务器
    metadata是基于wsgi的http服务，应该是可以水平扩展的，属于controller

    2、nova-api-os-compute 代码位置nova.cmd.api_os_compute
    api_os_compute就是osapi_compute
    参考路由服务的的代码
    点击(此处)折叠或打开
    # inherit all compute_api worker counts from osapi_compute
    if name.startswith('openstack_compute_api'):
        wname = 'osapi_compute'
    else:
        wname = name
    参考http://blog.csdn.net/xuriwuyun/article/details/37497619?utm_source=tuicool&utm_medium=referral
    参考摘要：nova-api中提供了很多服务API：ec2(与EC2兼容的API)，osapi_compute(openstack compute自己风格的API)
    总结：这个应该是和nova-compute通信的manager
    api_os_compute是基于wsgi的http服务，应该是可以水平扩展的，属于controller

    3、nova-api  代码位置nova.cmd.api
    这个服务是用来启动其他服务的（例如nova-api-os-compute、nova-api-metadata等）
    这个服务读取配置文件,启动配置中可以启动的服务,参考代码
    点击(此处)折叠或打开
    for api in CONF.enabled_apis:
        should_use_ssl = api in CONF.enabled_ssl_apis
        server = service.WSGIService(api, use_ssl=should_use_ssl)
        launcher.launch_service(server, workers=server.workers or 1)


    五、openstack-nova-common
    这个是所有nova包的的通用组件，nova的所有rpm包都会依赖openstack-nova-common，里面没有代码，只有公共配置文件和系统配置文件
    配置文件包括以下几类
    1、polkit-1相关控制配置
    2、nova-rootwrap的sudo配置,只有一条nova ALL = (root) NOPASSWD: /usr/bin/nova-rootwrap /etc/nova/rootwrap.conf *
    3、bash补全配置
    4、logrotate配置（滚日志配置）
    5、nova配置文件
        api-paste.ini  这个是nova的内部路由,没有跨版本和二次开发内容的话，不需要修改
        nova.conf      这个是nova的配置文件,具体如何配置看后面
        policy.json       默认权限配置文件,估计应该是全局一样的配置文件，没有二次开发可以不修改
        release           定义自己版本号的文件,不需要修改
        rootwrap.conf  应该是nova的shell工具的权限相关配置文件
        rootwrap参考http://blog.csdn.net/gaoxingnengjisuan/article/details/47102593
        参考摘要：
                    使用rootwrap的目的就是针对系统某些特定的操作，让非特权用户以root用户的身份来安全地执行这些操作   
                    所以rootwarp实现的具体步骤就是：
                    1 命令行解析
                    2 读取解析配置文件
                    3 加载过滤器文件中定义的所有命令行过滤对象
                    4 匹配到具体的命令行过滤对象
                    5 通过rootwrap的封装获取具体的命令行实现
                    6 派生一个子进程来执行命令行
                    实际上就是把命令行：
                    sudo nova-rootwrap /etc/nova/rootwrap.conf cat /etc/iscsi/initiatorname.iscsi
                    经过若干参数处理和匹配操作转化为：
                    sudo -u XXXX /bin/cat /etc/iscsi/initiatorname.iscsi之后，启动一个子进程来实现这个命令行的执行操作；
        总结：nova进行系统调用的时候(例如挂载iscsi硬盘)，使用sudo或去root权限执行命令过程中的权限验证工具。


    六、openstack-nova-compute
    1、入口代码
    点击(此处)折叠或打开
    CONF = nova.conf.CONF
    CONF.import_opt('compute_topic', 'nova.compute.rpcapi')
    LOG = logging.getLogger('nova.compute')

    def block_db_access():
        class NoDB(object):
            def __getattr__(self, attr):
                return self

            def __call__(self, *args, **kwargs):
                stacktrace = "".join(traceback.format_stack())
                LOG.error(_LE('No db access allowed in nova-compute: %s'),
                          stacktrace)
                raise exception.DBNotAllowed('nova-compute')

        nova.db.api.IMPL = NoDB()

    def main():
        config.parse_args(sys.argv)
        logging.setup(CONF, 'nova')
        utils.monkey_patch()
        objects.register_all()

        gmr.TextGuruMeditation.setup_autorun(version)

        if not CONF.conductor.use_local:
            block_db_access()
            objects_base.NovaObject.indirection_api = \
                conductor_rpcapi.ConductorAPI()
        else:
            LOG.warning(_LW('Conductor local mode is deprecated and will '
                            'be removed in a subsequent release'))

        server = service.Service.create(binary='nova-compute',
                                        topic=CONF.compute_topic,
                                        db_allowed=CONF.conductor.use_local)
        service.serve(server)
        service.wait()
    2、简要入口代码分析    
    通过use_local(配置文件中use_local=false)判断使用数据库还是使用Conductor（参考后面的第九部分的openstack-nova-conductor）
    默认使用Conductor，直接使用数据库的方式即将淘汰
    然后创建一个（rpc服务）服务，名字是CONF.compute_topic(配置文件中compute_topic=compute)，启动文件是nova-compute，不使用db(配置文件中use_local=false)
    顺便CONF.import_opt('compute_topic', 'nova.compute.rpcapi')， nova.compute.rpcapi就是nova代码对应的文件

    3、通过nova-compute rpm包的Requires确认nova-compute的功能
    Requires:         libvirt-python           通过libvirt，控制虚拟机
    Requires:         iptables iptables-ipv6   操作防火墙配置
    Requires:         iscsi-initiator-utils    iscsi挂载硬盘
    Requires:         python-cinderclient      块存储客户端
    Requires:         ipmitool                 管理物理机器（直接拿物理机当资源，不建立虚拟机）
    Requires:         bridge-utils             网桥管理
    其他Requires看到就大致知道还有什么功能了
    Requires bridge-utils因为有from nova import network
    network里包含有操作linux bridge的代码,所以nova-compute也Requires了bridge-utils

    直接用nova-network的话,nova-compute直接调用nova.network部分代码就可以操作网桥了
    使用neutron加linux网桥的话，需要安装openstack-neutron-linuxbridge
    使用neutron加openvswitch的话，需要安装openstack-neutron-openvswitch

    原因是，nova的控制网络部分独立出了neutron，对应的操作代码也应该从nova-compute独立出去
    但是旧的nova-network还没有删除，所以nova-compute依然保留了操作网络的功能

    另外
    由于qemu-kvm依赖qemu-img，所以rpm里没有Requires qemu-img
    代码里有调用qemu-img,也就是创建虚拟机的系统盘也是nova-compute的活

    4、总结
    nova-compute是装在物理机上用于管理虚拟机资源(严格来说是openstack资源，因为物理机也能作为资源)的程序，是个接受命令然后干活的程序
    由于nova-network还没完全移除，所以还保留了部分设置网络的功能
    不使用nova-network的话，设置网络的应该由对应的neutron agent组件来设置


    七、openstack-nova-network
    即将作废的组件,没有需求不关注


    八、openstack-nova-scheduler
    参考http://krystism.is-programmer.com/posts/64410.html
    参考摘要：nova-scheduler的功能是负责从多宿主机中调度最适合的宿主机生成云主机。即传入需要启动的云主机列表，nova-scheduler根据云主机的数量、参数等进行调度，选择合适的物理机
    总结:调度器自然是全局唯一的,从启动脚本看是个rpc服务


    九、openstack-nova-conductor
    rpm包里有这个服务启动文件和systemd配置文件
    参考http://lynnkong.iteye.com/blog/1813933
    参考摘要：在Grizzly版的Nova中，取消了nova-compute的直接数据库访问。大概两个原因：
     1. 安全考虑。因为compute节点通常会运行不可信的用户负载，一旦服务被攻击或用户虚拟机的流量溢出，则数据库会面临直接暴露的风险
     2. 方便升级。将nova-compute与数据库解耦的同时，也会与模式（schema）解耦，因此就不会担心旧的代码访问新的数据库模式。
    总结：G版中从openstack-nova-compute拆分出来的部分,负责从数据库获取记录并同rabbitmq通信的服务,启动代码看conductor是一个rpc服务


    十、openstack-nova-cells
    rpm包里有这个服务启动文件和systemd配置文件
    参考http://www.xuebuyuan.com/2029860.html
    参考摘要：  当启用了这个功能的时候，OpenStack Compute云中的主机会被分组，称作cell。cell的结构是树的形式。top-level级别的cell中的主机运行一个nova-api服务，但是可以没有nova-compute服务。每一个子cell应该运行常规OpenStack云计算中所有nova-*类型的服务，除了nova-api服务。我们可以把一个cell树结构看成一个正常的OpenStack
    总结：这个东西可以将openstack分层级，比如一个大楼有5楼，每一楼的openstack是一个cell（楼层级别，5个cell）,被一个顶层(整栋楼级别)cell管理
    顶层（整栋楼的级别）有nova-api、db、rabbitmq、nova-cell等
    每一楼（楼层级别）有自己独立的nova-api、rabbitmq、network、db、nova-compute等，
    上层（整栋楼的级别）通过nova-cell链接到下层（楼层级别）的rabbitmq
    如果OpenStack不需要分层，则不需要启动cells服务
    cells是一个rpc服务


    十一、openstack-nova-cert
    rpm包里有这个服务启动文件和systemd配置文件,一些脚本文件以及和证书相关的pem文件
    参考http://kiwik.github.io/openstack/2014/01/04/Nova-EC2%E6%8E%A5%E5%8F%A3token%E9%AA%8C%E8%AF%81%E6%B5%81%E7%A8%8B/
    参考摘要：我们用某一个用户登录Horizon的时候，左侧的“项目”tab中有一个“访问 & 安全”的链接，点击后页面右上角就会有一个”下载EC2凭证“的按钮，点击之后会从浏览器下载一个命名为${tenant_name}-x509.zip的zip包。使用哪个用户登录就会使用这个用户已经在keystone创建的access_key和secret_key, 需要注意的是，想要下载EC2凭证，nova-cert服务必须启动，因为需要nova-cert生成对应tenant的cert.pem和pk.pem
    顺便简单介绍下ec2
    http://docs.ocselected.org/openstack-manuals/kilo/admin-guide-cloud/content/instance-mgmt-ec2compat.html
    Euca2ools：和EC2 API互动的较流行的开源命令行工具。这在当EC2是通用API 或者从EC2云过渡到OpenStack的多云环境的情况下是非常便利的工具。更多信息，请参阅euca2ools 网站。
    boto：一个和亚马逊Web Service交互的python库。其可用于访问OpenStack，通过EC2兼容的API。
    总结：要用ec2就要有证书,证书需要通过openstack-nova-cert生成,后面的xvpvncproxy也需要这个服务
    总结：证书相关的服务,和EC2有关系,要用ec2,就要有一个openstack-nova-cert服务用于生成证书,从启动脚本看是个rpc服务


    十二、openstack-nova-novncproxy    openstack-nova-serialproxy   openstack-nova-spicehtml5proxy    openstack-nova-console   
    上述4相关的个放一起说,网上的内容不太清晰,尽量通过代码分析功能
    参考：http://www.choudan.net/2013/10/01/OpenStack-vnc-console%E9%85%8D%E7%BD%AE.html
    参考摘要:因为当你跟踪代码的时候，你会发现，从dashboard的请求，最终会落实到这个函数的调用上，get_vnc_console,该函数返回一个字典，表明可以连接到那个ip地址和端口去获取虚拟机的界面数据。
    port是从虚拟机的配置文件中读取出来的，host则是需要从配置文件中读取

    代码分析：novncproxy、serialproxy、spicehtml5proxy
    这三都是调用nova.cmd.baseproxy,而baseproxy则是websockify的封装
    websockify的说明
    At the most basic level, websockify just translates WebSockets traffic
    to normal socket traffic. Websockify accepts the WebSockets handshake,
    parses it, and then begins forwarding traffic between the client and
    the target in both directions.
    简单来说,新一代浏览器(html5浏览器)支持WebSockets,但是服务端程序并不支持WebSockets的包
    需要将WebSockets的包解出来转发给服务端程序
    novncproxy是代理novnc的端口
    serialproxy是代理serial的端口
    spicehtml5proxy是代理spicehtml5的端口
    我猜测上述服务都是有一台或者多台机器运行,用户web中选择对应方式会分配到一个代理机器,链接到代理机器后,将目标机器的信息发送过去,转发服务自动代理到对应机器的vnc ip端口
    也可能每台资源机器运行一个代理实例,专门代理本机转发本机vnc
    更有肯定是openstack集群中有多个horizon,每个horizon带一个代理
    反正肯定不能只有一个,不然那么多vnc代理都给python写的来转发,数据量多了肯定卡
    至于用哪个,我估计原生的spicehtml5proxy可能支持还没那么多,最广泛的估计还是novncproxy

    openstack-nova-console
    组件1:xvpvncproxy
    xvpvncproxy是一个wsgi_server,xvpvncproxy注册名字是XCP VNC Proxy,应该是不使用代理直接通过web页面直接连接到vnc,查资料支持专门写的java客户端,估计类似HP的ilo的方式
    参考http://cwtea.blog.51cto.com/4500217/1348218/
    参考摘要:nova-xvpnvncproxy 守护进程，通过vnc连接访问正在运行的实例代理，支持专门设计的Openstack的java客户端，nova-cert 守护进程，管理x509证书
    组件2:consoleauth服务
    这个是专门给console授权的提供token的
    这里可以看出网上的“Nova-console 已经弃用，被 nova-xvpnvncproxy取代”应该是错误的,应该是M版本中默认的console是xvpvncproxy。
    组件3:%{_bindir}/nova-console*
    这个看上去像shell的控制台

    openstack-nova-doc
    这个不用说了,全是文档
