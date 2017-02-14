---
layout: post
title:  "pacemaker的clone配置说明以及RHCS集群故障处理"
date:   2016-08-16 15:05:00 +0800
categories: "troubleshooting"
tag: ["rhcs", "drbd", "pacemaker", "linux", "集群"]
---

* content
{:toc}


[clone(克隆)配置文档地址](http://clusterlabs.org/doc/en-US/Pacemaker/1.1-plugin/html-single/Pacemaker_Explained/index.html#_clone_options)

##### 克隆资源说明
    clone  在一个Active/Active模式中，某些资源在群集的多个节点上同时运行。为此,必须将资源配置为克隆资源。可以配置为克隆资源的资源包括 STONITH 和群集文件系统(如OCFS2)。
    如有resource agent支持,则可以克隆任何资源。克隆资源的配置甚至也有不同,具体取决于资源驻留的节点。
    资源克隆有三种类型,有三种,匿名克隆(默认)、全局唯一克隆、多状态克隆


## 克隆资源特有参数

##### globally-unique 克隆状态通过这个参数来控制

1. 匿名克隆：参数globally-unique=false。这是最简单的克隆类型。这种克隆类型在所有位置上的运行方式都相同。因此,每台计算机上只能有一个匿名克隆实例是活动的。
2. 全局唯一克隆：参数globally-unique=true。这些资源各不相同。一个节点上运行的克隆实例与另一个节点上运行的实例不同,同一个节点上运行的任何两个实例也不同。
3. 状态克隆：作为参数出现在ms/master中。这些资源的活动实例分为两种状态:主动和被动，也称主和从。状态克隆可以是匿名克隆也可以是全局唯一克隆。

##### clone-max: 在集群中最多能运行多少份克隆资源，默认和集群中的节点数相同；

##### clone-node-max：每个节点上最多能运行多少份克隆资源，默认是1；
---

### 上面几个参数我们目前还用不到不用管,下面是常用参数

##### notify：当成功启动或关闭一份克隆资源，要不要通知给其它的克隆资源，可用值为false,true(默认状态英文文档上的没注意,反正主从克隆状态需要显式指定为true)

##### ordered：Should the copies be started in series (instead of in parallel).  为true时,克隆资源是串行启动，而非一起(parallel)启动,可用值为false,true；默认值是false

    这个ordered和constraint的order不是一个东西,这个是当前resource控制自身clone之间是否有序启动的
    和其他resource没有关系,constraint里的order是控制不同resource顺序的

##### interleave：下面是原文,这个参数比较重要,一定要仔细理解,没有这个参数的时候默认是false

    Changes the behavior of ordering constraints (between clones/masters)
    so that instances can start/stop as soon as their peer instance has (rather
    than waiting for every instance of the other clone has). Allowed values: false, true

    修改顺序约束在clone/masters中的表现
    使得实例可以start/stop自己的实例而不用等待所有其他clone


##### 默认false,这个设置为false的时候,constraint的order顺序的受到其他节点的影响,为true不受其他节点影响.比如说php-fpm需要自己节点的mount完就能启动,如果为默认的false,那么需要其他集群节点都mount完后,当前结点php-fpm才会启动,这个参数是设置在被约束的资源上的,比如

    dlm上的设置interleave是没有用,因为dlm没有被约束,要让drbd在dlm都启动以后才启动
    所以,我们需要将interleave=false设置在ms-drbd上（但是会造成其他问题,下面有描述）


##### 之前打算实现一个约束, 让所有drbd节点设置Primary状态后mount才能进行,于是mount设置interleave=false, 添加约束promote ms-drbd then start fs_gfs2-clone,结果

    单独stop一个节点然后start它这个节点必然出现mount不会等待drbd设置Primary就开始mount
    而且drbd会在mount失败后才去设置Primary

##### 于是我根据英文说明猜测,interleave只关心约束对象的start/stop状态

    因为当interleave为false的时候
    被约束对象(mount)只关心约束对象(drbd)是否在所有节点都start
    所以当前结点的promote约束(promote约束内部肯会隐式的添加start约束)被忽略了
    也就是说mount资源interleave=false的全局约束覆盖了当前结点的约束

##### 后来又发现ms-drbd设置interleave=false会出现

    单独stop一个节点然后start这个节点,drbd必然脑裂都进入standby状态
    即使drbd的盘没有写入过任何数据,也会出现这个问题

#### 最后在多次测试后终于发现

    当interleave为true时, 停止一个节点,再开启的时候,集群会将所有节点资源关闭,再按照约束启动所有资源
    当interleave为false时, 停止一个节点,再开启的时候,只有这个节点的资源被启动,出现了各种约束失效


## 所以,interleave为flase的时候,drbd/gfs2双写系统不要尝试单独关闭节点,以免出现异常


---

## resource参数

##### timeout   start stop monitor都有的参数,这个不解释了

##### monitor   特有参数

1. interval  执行monitor的间隔
2. on-fail： monitor检查失败时执行的操作。允许的值：

```
ignore：假装资源没有失败。
block：不对资源执行任何进一步操作。
stop：停止资源并且不在其他位置启动该资源。
restart：停止资源并（可能在不同的节点上）重启动。(默认值)
fence：关闭资源失败的节点 (STONITH)。(默认值,有STONITH的情况下)
standby：将所有资源从资源失败的节点上移走。
上面是抄的,我记得有些默认值是不对的,具体看对应版本的英文文档
```

##### 特殊参数 Options
    Options inherited from primitive resources: priority, target-role, is-managed
    克隆资源从原始资源继承的Options有priority, target-role, is-managed,这里的Options不是op monitor设置的,是指下面的
    Resource Options Options are used by the cluster to decide how your resource should behave and can be easily set using the --meta option of the crm_resource command.
---

## 集群故障处理


#### drbd资源报错显示资源unmanage
    Failed actions:
    drbd_stop_0 on node1 'not configured'
这个错误很容易被误解为action没有配置,实际错误可以在/var/log/messages找到错误信息

    ERROR: you really should enable notify when using this RA (or set ignore_missing_notifications=true)

这个错误是/usr/lib/ocf/resource.d/linbit/drbd抛出的,查看这个脚本我们可以大致知道和参数OCF_RESKEY_CRM_meta_notify、OCF_RESKEY_CRM_meta_notify_start_uname有关

    通过上述方式,类似错误都可以通过这样的方式定位(用debug-start执行有时候定位不到)
需要通过syslog来debug

通过syslog debug[参考这里](http://blog.chinaunix.net/uid-23504396-id-3786530.html)

#### pcs status里显示集群节点online但是/var/log/cluster/corosync.log不停报错

    error: remote_op_done:	Operation reboot of node2 by for stonith_admin.cman
    notice: tengine_stonith_notify:	Peer node2 was not terminated (reboot) by node1
我们在/var/log/messages看到同样错误以及下面额外信息

    cman_get_cluster error -1 112
    fence node1 dev 0.0 agent none result: error from ccs

fence相关错误是因为没有fence设备不能强制重启节点,但主要原因还是因为集群节点通信故障,cman_get_cluster error,集群并没有正常运行

    我们pcs status里显示onlien其实并不是表示集群节点正常运行
    因为pcs是通过pcs自身的web程序去post其他节点的pcs web接口获取的
    我估计pcs的守护进程检查了下集群的进程是否在正常运行,如果进程正常就就显示节点online
    pcs并没有检查集群的通信内容

#### 集群节点通信故障

##### 1. 首先集群节点通信故障的时候pcs是能正常设置集群的,但是会出现一些异常表现比如删除资源后资源还在,需要cleanup

##### 2. pcs可以启动集群但是关闭集群会被阻塞只能kill

##### 集群节点通信故障一般就这几个原因

1. 集群配置错误,如果我们是通过pcs来配置集群,那么集群的配置文件一般是没什么错误
2. 防火墙,这个比较容易定位
3. 组播通信故障,因为corosync默认多播通信,一些特殊的系统设置和系统环境会影响到这个通信


    1. LVS节点的arp_announce设置导致组播通信故障
    如果使用了
    net.ipv4.conf.all.arp_ignore = 1  # 这个应该对组播通信应该没影响
    net.ipv4.conf.all.arp_announce = 2
    有可能造成组播通信故障,就出现了pcs检查节点正常但是实际集群并不能正常通信
    我需要先设置并执行sysctl -p来还原默认配置
    net.ipv4.conf.all.arp_ignore = 0
    net.ipv4.conf.all.arp_announce = 0
    然后只设置LVS对应网卡比如eth0
    net.ipv4.conf.eth0.arp_ignore = 1
    net.ipv4.conf.eth0.arp_announce = 2

    2. 使用云服务器组播通信故障
    云服务器上因为使用虚拟网络,为了节约网咯资源一般无法进行多播通信
    查看配置文件
    man corosync.conf
    transport
           This  directive controls the transport mechanism used.
           If the interface to which corosync is binding is an RDMA interface
           such as RoCEE or Infiniband, the "iba" parameter may be specified.
           To avoid  the  use of  multicast entirely,
           a unicast transport parameter "udpu" can be specified.
           This requires specifying the list of members
           that could potentially make up the membership before deployment.
           The default is udp.  The transport type can also be set to udpu or iba.

    我们可以把通信方式改成udp,这样云服务上集群通信就没问题了
    修改/etc/cluster/cluster.conf,修改transport

#### gfs2无法mount

    mount point already used or other mount in progress
    error mounting lockproto lock_dlm

解决这个问题花了我好几个钟头

上述问题随便google一下都说是bug,其实不是
上述问题最容易在只有2个节点又使用dlm锁的集群中出现,因为

    当只有2个节点的时候,没有第三方作为仲裁,所以当节点通信故障的时候,都会认为是对方故障
    当上述情况出现的时候,各自节点的dlm锁都以自己为初始节点创建gfs2的table
    当集群正常通信以后,每个节点都的dlm锁中都有gfs2的table,但是又不是一致的


知道原因以后解决起来就很简单了
1. 既然是锁的问题,重置锁就可以了
2. dlm是用于同步的,对gfs2来说dlm锁的数据自然也是可以丢失的,所以重置不会影响到gfs2的数据
3. gfs2的dlm锁配置肯定是存在的,因为gfs2的配置文件中没有对应配置信息,那么锁是配置在文件系统里的

man 一下gfs2_tool, 可以知道gfs2_tool可以修改gfs2的table和proto

    [root@youzhai-web1 dlm]# gfs2_tool sb /dev/drbd0 proto
    current lock protocol name = "lock_dlm"
    [root@youzhai-web1 dlm]# gfs2_tool sb /dev/drbd0 table
    current lock table name = "web_gfs:gfs2_drbd"

我们可以临时修改gfs2使用的dlm表来让其中一个节点正常工作,但是可能只有一个节点有效.
重启所有节点集群是重置dlm的最好方法,但是这里有个坑

1. 节点无法关闭只能kill
2. 节点因为dlm锁异常umount会被阻塞(最近测试发现应该不是dlm造成的,是drbd造成的)
3. 因为无法umount导致reboot命令没效果
4. 没注意到reboot无效,重启后节点dlm信息还是混乱的(这是个大坑,解决问题的时间都在这里了)

所以重启节点有可能需要强制重启
强制重启使用以下命令,[参考来源](http://su-junjie.blog.163.com/blog/static/35123974201331324419480/)

    # 向sysrq文件中写入1, 开启SysRq功能
    echo 1 > /proc/sys/kernel/sysrq
    # 通过SysRq强制重启
    echo b > /proc/sysrq-trigger
    # 'b' —— 将会立即重启系统，并且不会管你有没有数据没有写回磁盘，也不卸载磁盘，而是完完全全的立即关机
    # 'o' —— 将会关机
    # 's' —— 将会同步所有以挂在的文件系统
    # 'u' —— 将会重新将所有的文件系统挂在为只读属性

强制重启很可能造成文件系统异常,需要fsck.gfs2(虽然没有操作错误,但是回想自己到底用的是mkfs还是fsck的时候还是有点慌)

在不启动集群的情况下启动drbd,对drbd的设备分区进行fsck,然后关闭drbd

这时候再启动集群,dlm锁就正常了,gfs2也能正常mount了

# 总结

## 启用dlm锁的双节点集群,节点绝对不要自动启动,一定要确保节点通信正常后再启动,否则麻烦死你
