---
layout: post
title:  "OpenStack Mitaka从零开始 虚拟机实例通过openvwitch通外网的简要过程,dvr与dvr-snat的区别"
date:   2017-02-05 15:05:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}


#### 前言：下面所说的veth pair都是只veth pair这种方式而不是指linux的veth pair. openstack默认使用openvwitch来实现veth pair.
但是早期资料表示openvwitch实现的veth pair不支持tc,所以要流控需要配置为使用linux自带的veth pair.

参考[链接](http://bingotree.cn/?p=708&utm_source=tuicool&utm_medium=referral)

---

neutron配置为dvr-snat,这样无外网ip的实例的才能访问外网

1.网络分配(暂时介绍和dhcp相关的东西,所有网络不开DHCP,但是虚拟机要或去ip必须有dhcp服务)

    没有实际的外网IP,所以直接用一个172的网段当外网
    这里我们使用172.30.0.0/24
    虚拟机内部IP段为
    我们使用192.168.1.0/24  

172.30.0.0/24由于是外网(create network的时候external network打勾),需要由cloud admin创建并设置共享(share打勾)。
创建子网的时候网关设置为对应vlan的网关,我这里用的是172.30.0.1,范围要错开交换机上的vlan interface ip

192.168.1.0/24网段由普通用户创建网络并分配子网,需要通过内网路由到外网,所以需要网关,使用192.168.1.1


2.路由创建

    和ulcoud、qcloud一样,虚拟机中只有内网IP,看不到外网IP
    通过route,外网IP的访问转发到内网IP上
    所以需要创建路由将内网网段和外网网段打通
    其实,就是通过namespace、ovs、iptable实现了三层交换机的路由功能
    所以,折腾这里之前,建议先通过物理设备联系下vlan划分、交换机上做vlan路由通子网的操作

---

### 为了更容易理解,我们先看无外网IP但是可以访问外网的情形(只能访问外网,不能被外部网络访问、即只出不进)

首先看ovs网桥

    Bridge br-int
        fail_mode: secure
        Port "qr-e0711d58-58"         192.168.1.1
            tag: 1
            Interface "qr-e0711d58-58"
                type: internal
        Port "tapcf6bc2eb-b1"		  192.168.1.3
            tag: 1
            Interface "tapcf6bc2eb-b1"
        Port "sg-8f551a28-9e"         192.168.1.4
            tag: 1
            Interface "sg-8f551a28-9e"
                type: internal
    Bridge br-ex
        Port br-ex
            Interface br-ex
                type: internal
        Port "enp3s0f1"                 物理网卡
            Interface "enp3s0f1"
        Port "qg-d991dda3-eb"           172.30.0.10
            Interface "qg-d991dda3-eb"
                type: internal

路由的snat空间

    Port "sg-8f551a28-9e"         192.168.1.4	network:router_centralized_snat
    Port "qg-d991dda3-eb"         172.30.0.10	network:router_gateway

    # snat空间的路由就是default gw 172.30.0.1

路由的qrouter空间，这个空间的路由策略有特殊处理

    # 接口
    Port "qr-e0711d58-58"         192.168.1.1	network:router_interface_distributed   虚拟机网关地址
    Port "tapcf6bc2eb-b1"		  192.168.1.3	compute:nova       虚拟机IP  

    # 路由策略
    [root@openstack ~]# ip netns exec qrouter-a74892a9-8fea-4495-ad58-4d1244065b9c ip rule show
    0:      from all lookup local
    32766:  from all lookup main
    32767:  from all lookup default   # default路由表是空表
    3232235777:     from 192.168.1.1/24 lookup 3232235777   # main路由表没有被匹配的是后,走这个路由表

    # 3232235777路由表信息,就一条,网关是192.168.1.4
    default via 192.168.1.4 dev qr-e0711d58-58


重点在于理解192.168.1.4与192.168.1.1,为什么有了192.168.1.1还会需要一个192.168.1.4.

qrouter空间要通向snat,再通过172.30.0.0段穿出去,需要打通两个namespace,
192.168.1.4和192.168.1.1就是打通两个命名空间veth pair(openstack默认用ovs实现这个功能,也就是他俩都在一个Bridge上).
veth pair两端都需要IP,所以我们不仅看见了这头的网关IP 192.168.1.1,还另外占用了一个192.168.1.4的IP.
同样外网IP虽然不是veth pair,但是逻辑类似,这一头有一个172.30.0.10, 和外网的网关172.30.0.1是可达的.
其实就是类似三层交换机做路由,需要两边都有相同网段的vlan interface ip

路由过程如下

    在没有外网的情况下,我们虚拟机的网关是192.168.1.1,数据包发往192.168.1.1
    defalut路由表中没有被匹配到出去的下一条地址,路由策略使用下一个路由表3232235777
    这个路由表只有一条默认网关为192.168.1.4的路由
    也就是说我们的对外请求先发到192.168.1.1,然后被qr-e0711d58-58发到192.168.1.4
    这样出去的数据包就到了snat空间,最后通过snat空间的网关172.30.0.1发到外部


实现上面功能,操作步骤为
1、创建路由,选172.30.0.0/24段
2、进入路由中增加接口,选择内网192.168.1.0/24段,网关不用填,自动使用子网创建时的网关

---

### 现在我们再看有外网IP的情形

直接给虚拟机分配外网IP的方式我们就不讨论了,因为实际业务中虚拟机实例绑定IP的做法非常不灵活。
从这里可以看出.阿里云的虚拟机中有2个网卡,一个网卡带有外网IP是非常落后的做法(卸载外网IP需要虚拟机实例的系统配合)。

我们来看普遍做法——内网通过路由链接到外网,然后为虚拟机实例绑定一个浮动IP.前面我们已经完成了路由设置,再为虚拟机绑定一个浮动IP就可以了.

我们来看看绑定浮动IP后的ovs和namespace,因为重建了一次路由,所以IP有些变化,原理和前面一样。


当前内网IP 192.168.1.6, 外网IP 172.30.0.16


ovs网桥

    Bridge br-int
        Port "sg-72ac01bb-72"
            tag: 9
            Interface "sg-72ac01bb-72"
                type: internal
        Port "qr-8868c230-55"
            tag: 9
            Interface "qr-8868c230-55"
                type: internal
        Port "tapc8a38809-f6"
            tag: 9
            Interface "tapc8a38809-f6"


    Bridge br-ex
        Port br-ex
            Interface br-ex
                type: internal
        Port "qg-cae8db5d-f4"
            Interface "qg-cae8db5d-f4"
                type: internal
        Port "enp3s0f1"
            Interface "enp3s0f1"
        Port "fg-f4980cc3-de"
            Interface "fg-f4980cc3-de"
                type: internal


qroute空间

    qr-8868c230-55 192.168.1.1/24
    rfp-a74892a9-8 169.254.106.114/31


    # 路由信息
    [root@openstack ~]# ip netns exec qrouter-a74892a9-8fea-4495-ad58-4d1244065b9c ip rule show
    0:      from all lookup local
    32766:  from all lookup main
    32767:  from all lookup default
    57481:  from 192.168.1.6 lookup 16
    3232235777:     from 192.168.1.1/24 lookup 3232235777

    # 来自192.168.1.6,全部从通过rfp-a74892a9-8,从169.254.106.115出去
    # 也就是直接穿到浮动IP空间的fpr-a74892a9-8
    # 也就是说不走到snat空间
    [root@openstack ~]# ip netns exec qrouter-a74892a9-8fea-4495-ad58-4d1244065b9c ip route show table 16
    default via 169.254.106.115 dev rfp-a74892a9-8

    # default table的路由
    169.254.106.114/31 dev rfp-a74892a9-8  proto kernel  scope link  src 169.254.106.114
    192.168.1.0/24 dev qr-8868c230-55  proto kernel  scope link  src 192.168.1.1

snat空间

    qg-cae8db5d-f4   172.30.0.10/24
    sg-72ac01bb-72   192.168.1.8

    default via 172.30.0.1 dev qg-cae8db5d-f4
    172.30.0.0/24 dev qg-cae8db5d-f4  proto kernel  scope link  src 172.30.0.10
    192.168.1.0/24 dev sg-72ac01bb-72  proto kernel  scope link  src 192.168.1.8


浮动IP空间

    fpr-a74892a9-8  169.254.106.115/31
    fg-f4980cc3-de  172.30.0.11/24

    default via 172.30.0.1 dev fg-f4980cc3-de
    169.254.106.114/31 dev fpr-a74892a9-8  proto kernel  scope link  src 169.254.106.115
    172.30.0.0/24 dev fg-f4980cc3-de  proto kernel  scope link  src 172.30.0.11
    172.30.0.16 via 169.254.106.114 dev fpr-a74892a9-8

iptable的内容就不贴了,可以看到和之前无外网IP单通外网的区别在于

    1.多出了个浮动ip空间
    2.多了一对veth pair  fpr-a74892a9-8与rfp-a74892a9-8
    3.qroute空间多了一条路由策略和对应的路由表
    4.有浮动ip的情况下,路由策略不再走snat空间,而是通过浮动ip空间到达br-ex


## 总结
1.dvr-snat是给无外网IP的虚拟机访问外网用的。

2.如果不分配外网虚拟机不需要访问外网,配置为dvr即可（也就是不走snat空间到br-ex）。

3.openstack中虚拟机访问外网,需要经过多个网关、通过iptable多次nat,如果使用ovs,性能堪忧。
