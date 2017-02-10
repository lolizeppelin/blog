---
layout: post
title:  "OpenStack Mitaka从零开始 openvwitch学习,流表解读,vxlan隧道"
date:   2017-02-05 15:05:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}



[Opevswitch总体架构](http://www.cnblogs.com/popsuper1982/p/5848879.html#3614533)

[IBM文档中心参考文章](http://www.ibm.com/developerworks/cn/cloud/library/1401_zhaoyi_openswitch/)

### 一些名词解释

1.同一主机上的OVS中可以创建多个网桥（即多个datapath实例）

2.接口设备(Interface、虚拟网卡、物理网卡等)通过port连接到ovs。在通常情况下，Port和 Interface是一对一的关系,只有在配置Port为bond模式后，Port和Interface是一对多的关系。

3.bridge间可以通过patch ports互联(不需要Interface).

ovs中的端口有多种类型

    Normal(也就是netdev ports?)  
    用户可以把操作系统中的网卡绑定到ovs上，ovs会生成的同名端口处理这块网卡进出的数据包
    也就是说,物理接口和bridge相连的端口类型是Normal/netdev ports,ovs-vsctrl show的时候不显示的就是这类

    internal(与虚拟网卡链接的port)
    1.当ovs创建一个新网桥时，默认会创建一个与网桥同名的Internal Port(以及虚拟Interface)
    2.虚拟网卡，链接到bridge的时候创建同名的Internal Port

    patch
    当机器中有多个ovs网桥时，可以使用Patch Port把两个网桥连起来。Patch Port总是成对出现，分别连接在两个网桥上，在两个网桥之间交换数据。

    tunnel(这个类型现在拆分为vxlan和gre)
    隧道端口是一种虚拟端口，支持使用gre或vxlan等隧道技术与位于网络上其他位置的远程端口通讯。

所以ovs-vsctl show和ovs-vsctl list-ports里显示的都是端口名,只是名字和网卡名相同罢了

下面是具体例子

    Bridge br-tun
        fail_mode: secure
        Port "vxlan-0a0a0033"
            Interface "vxlan-0a0a0033"
                type: vxlan
                options: {df_default="true", in_key=flow, local_ip="10.10.0.52", out_key=flow, remote_ip="10.10.0.51"}
        Port patch-int
            Interface patch-int
                type: patch
                options: {peer=patch-tun}
        Port "enp2s0f1"
            Interface "enp2s0f1"
        Port br-tun
            Interface br-tun
                type: internal
    Bridge br-int
        Port patch-tun
            Interface patch-tun
                type: patch
                options: {peer=patch-int}
        Port br-int
            Interface br-int
                type: internal
        Port "tap4c8a8752-e2"
            tag: 1
            Interface "tap4c8a8752-e2"
        Port "tap8c6f222b-1e"
            tag: 1
            Interface "tap8c6f222b-1e"
                type: internal

1.br-int中的br-int是和网桥同名端口
2.br-int中的tap网卡是和虚拟机网卡相连的端口
3.br-tun中的patch-int和br-int的patch-tun是一对path用于两个网桥互联
4.br-tun中的vxlan-0a0a0033是vlan隧道端口,ifconfig中可以看到vlan网卡叫vxlan_sys_4789,不一定要和端口名字相同

---

OpenvSwitch的组成[参考](http://blog.csdn.net/lizheng2300/article/details/54582310)

    （1）OVS内核空间（包含Datapath和流表）

    Datapath：主要负责实际的数据分组处理，把从接收端口收到的数据包在流表中进行匹配，并执行匹配到的动作。它同时与vswitchd和流表保持关联，使OVS上层可以对数据分组处理进行控制。
    Flow table(流表)：流表中存储着分组处理的依据--流表项，每个Datapath 都和一个流表关联，当Datapath接收到数据之后，OVS会在流表中查找可以匹配的 flow（流表项），执行对应的操作,例如转发数据到另外的端口。同时还与vswitchd进程上下关联，是OVS上层对底层分组处理过程进行管理的接口。

    （2）OVS用户空间（vswitchd和ovsdb）
    vswitchd：守护程序，实现交换功能，和Linux内核模块一起，实现基于流的交换flow-based switching，负责检索和更新数据库信息，并根据数据库中的配置信息维护和管理OVS。Vswitchd可以配置一些列特性：基于MAC地址学习的二层交换、支持IEEE 802.1Q VLAN 、端口镜像、sFlow监测、连接OpenFlow控制器等。Vswitchd也可以通过Netlink协议与内核模块Datapath直接通信。
    ovsdb-server：轻量级的数据库服务，主要保存了整个OVS的配置信息和数据流信息，ovsdb-server直接管理ovsdb，与ovsdb通信进行数据库的增删改查操作。Vswitchd可以通过socket与ovsdb-server进行通信，用于查询和更新数据库信息，或者在检索数据库信息后做出首个数据包的转发策略。

    （3）OVS管理工具
    ovs-dpctl：管理OVS Datapath的实用工具，用来配置交换机内核模块，控制数据包的转发规则。可以创建、修改和删除Datapath。
    ovs-vsctl：查询和配置OVS数据库的实用工具，用于查询或修改vswitchd的配置信息，该工具会直接更新ovsdb数据库。
    ovs-appctl：主要是向OVS守护进程发送命令的工具，一般用不上。
    ovs-ofctl：用来控制OVS作为OpenFlow交换机工作时的各种参数，它是OVS提供的命令行工具。在没有配置 OpenFlow控制器的模式下，用户可以使用 ovs-ofctl命令通过 OpenFlow协议去连接 OVS，创建、修改或删除 OVS中的流表项，并对 OVS的运行状况进行动态监控。
    ovs-pki：OpenFlow交换机创建和管理公钥框架。
    ovs-tcpundump：tcpdump的补丁，解析 OpenFlow的消息。



常规列

    Cookie：流规则标识。
    Priority:流表项的优先级，数字越大优先级越高，范围是：0~7。

条件列

    table：流表项所属的table编号。
    in_port:传递数据包的端口的 OpenFlow 端口编号
    dl_vlan:数据包的 VLAN Tag 值，范围是 0-4095，0xffff 代表不包含 VLAN Tag 的数据包
    dl_src/dl_dst:匹配源或者目标的 MAC 地址,加斜线表示用通配符匹配
                  01:00:00:00:00:00 多播
                  01:00:00:00:00:00/01:00:00:00:00:00  通配所有多播(包括广播)
                  00:00:00:00:00:00/01:00:00:00:00:00  通配所有单播
                  ff:ff:ff:ff:ff:ff  精确匹配 (相当于省略子网掩码,这个应该一般是arp才发这个地址).
                  00:00:00:00:00:00  通配所有
    dl_type:匹配以太网协议类型
            dl_type=0x0800 代表 IPv4 协议
            dl_type=0x086dd 代表 IPv6 协议
            dl_type=0x0806 代表 ARP 协议
            以及其他协议
    arp_spa/arp_tpa: 分别匹配匹配来源/目标IP地址


计数器列

    duration：流表项创建持续的时间（单位是秒）。
    n_packets：此流表项匹配到的报文数。
    n_bytes：此流表项匹配到的字节数。
    idle_age：此流表项从最后一个匹配的报文到现在空闲的时间。
    hard_age：此流表项从最后一次被创建或修改到现在持续的时间。


动作列

    normal: Subjects the packet to the device's normal L2/L3 procesing.
    IN_PORT: 将包裹从原路扔回
    output(参数port): 输出数据包到指定的端口。port 是指端口的 OpenFlow 端口编号
    mod_vlan_vid(参数vlan id): 修改数据包中的 VLAN tag
    mod_vlan_pcp(参数vlan id)：Modifies the VLAN priority on a packet.
    strip_vlan(无参数): 移除数据包中的 VLAN tag
    mod_dl_src/ mod_dl_dest(参数MAC): 修改源或者目标的 MAC 地址信息
    mod_nw_src/mod_nw_dst(参数IP): 修改源或者目标的 IPv4 地址信息
    load(参数value−>dst[start..end]): 写数据到指定的字段
    resubmit(参数([port],[table])):  Re-searches this OpenFlow flow table (or the table whose number is specified by table)
                        with the in_port field replaced by port (if port is specified)
    set_tunnel/set_tunnel64(参数tun id)  If outputting to a port that encapsulates the packet in a tunnel and supports an identifier (such as GRE), sets the identifier to id.
    mod_vlan_vid:vlan_vid
            Modifies the VLAN id on a packet.
    move:src[start..end]−>dst[start..end]
            Copies the named bits from field src to field dst. src and dst must be NXM field names as defined in nicira−ext.h, e.g. NXM_OF_UDP_SRC or NXM_NX_REG0.
            Examples: move:NXM_NX_REG0[0..5]−>NXM_NX_REG1[26..31] copies the six bits numbered 0 through 5, inclusive, in register 0 into bits 26 through 31, inclusive; move:NXM_NX_REG0[0..15]−>NXM_OF_VLAN_TCI[] copies the least significant 16 bits of register 0 into the VLAN TCI field.
    load:value−>dst[start..end]
            Writes value to bits start through end, inclusive, in field dst.
            Example: load:55−>NXM_NX_REG2[0..5] loads value 55 (bit pattern 110111) into bits 0 through 5, inclusive, in register 2.

### 上面的列只是部分列说明,详细可以看[文档](http://openvswitch.org/support/dist-docs/ovs-ofctl.8.txt),也就是man 8 ovs-ofctl

### 现在我们来分析下流表

    ===============================这里是mac对应IP================================================================================================================
    fa:16:3e:2f:e7:eb     192.168.1.1  子网网关

    fa:16:3e:d3:4f:c1     192.168.1.6  虚拟机1  
    fa:16:3e:97:bd:37     192.168.1.9  dhcp服务接口IP
    fa:16:3e:d5:8e:54     192.168.1.8  SNAT地址(这个是启用的drv-snat才有的)

    fa:16:3e:f4:18:f5     192.168.1.12 虚拟机2   当前流表所在虚拟机

    ===============================这里是端口=====================================================================================================================
    enp2s0f1    port 1
    patch-int   port 2
    vxlan-0a0a0033  port 3

    ===============================这里开始是流表==================================================================================================================
    table=0, priority=1,in_port=2 actions=resubmit(,1)      port=2进来的走表1,也就是从内部虚拟机发来的包走1

    table=0, priority=1,in_port=3 actions=resubmit(,4)      port=3进来的走表4,也就是从外部收到的包走4

    table=0, priority=0 actions=drop
    ===============================================================================================================================================================
    table=1, priority=3,arp,dl_vlan=1,arp_tpa=192.168.1.1 actions=drop          匹配到目标地址是网关的包丢弃(因为用了dvr,所以每个l3节点有自己的192.168.1.1)

    table=1, priority=2,dl_vlan=1,dl_dst=fa:16:3e:2f:e7:eb actions=drop         和上个规则目的一样,匹配的是192.168.1.1的MAC地址

    table=1, priority=1,dl_vlan=1,dl_src=fa:16:3e:2f:e7:eb actions=mod_dl_src:fa:16:3f:e5:07:99,resubmit(,2)   从192.168.1.1来的包修源mac为fa:16:3f:e5:07:99,这mac是啥？ 走2

    table=1, priority=0 actions=resubmit(,2)                                    上述规则未匹配, 走表2
    ===============================================================================================================================================================
    table=2, priority=1,arp,dl_dst=ff:ff:ff:ff:ff:ff actions=resubmit(,21)                  ARP协议广播包走表21

    table=2, priority=0,dl_dst=00:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,20)    非ARP协议广播包走表20

    table=2, priority=0,dl_dst=01:00:00:00:00:00/01:00:00:00:00:00 actions=resubmit(,22)    所有单播包走表22
    ===============================================================================================================================================================
    table=3, priority=0 actions=drop              走到表3的都直接丢弃
    ===============================================================================================================================================================
    table=4, priority=1,tun_id=0x138f actions=mod_vlan_vid:1,resubmit(,9)       匹配到tun_id=0x138f,修改vlan id为1,走表9

    table=4, priority=0 actions=drop                                            没匹配到上面的tun_id=0x138f,直接丢弃
    ===============================================================================================================================================================
    table=6, priority=0 actions=drop             走到表6的都直接丢弃
    ===============================================================================================================================================================
    table=9, priority=1,dl_src=fa:16:3f:45:5e:f9 actions=output:2       来源地址为fa:16:3f:45:5e:f9的走表2    这个地址是啥？

    table=9, priority=0 actions=resubmit(,10)                           未匹配到来源地址fa:16:3f:45:5e:f9走表10  
    ===============================================================================================================================================================
    table=10,priority=1 actions=learn(table=20,hard_timeout=300,priority=1,NXM_OF_VLAN_TCI[0..11],NXM_OF_ETH_DST[]=NXM_OF_ETH_SRC[],load:0->NXM_OF_VLAN_TCI[],load:NXM_NX_TUN_ID[]->NXM_NX_TUN_ID[],output:NXM_OF_IN_PORT[]),output:2                                                                  这条看不懂
    ===============================================================================================================================================================
    table=20, priority=2,dl_vlan=1,dl_dst=fa:16:3e:d3:4f:c1 actions=strip_vlan,set_tunnel:0x138f,output:3     目标mac是fa:16:3e:d3:4f:c1,也就是对面的192.168.1.6,设置tun id,去除vlan标记,走3号出口

    table=20, priority=2,dl_vlan=1,dl_dst=fa:16:3e:97:bd:37 actions=strip_vlan,set_tunnel:0x138f,output:3     同上

    table=20, priority=2,dl_vlan=1,dl_dst=fa:16:3e:d5:8e:54 actions=strip_vlan,set_tunnel:0x138f,output:3     同上

    table=20, priority=0 actions=resubmit(,22)           未匹配到上述规则,走22
    ===============================================================================================================================================================
    table=21, priority=1,arp,dl_vlan=1,arp_tpa=192.168.1.6 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:fa:16:3e:d3:4f:c1,load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0xfa163ed34fc1->NXM_NX_ARP_SHA[],load:0xc0a80106->NXM_OF_ARP_SPA[],IN_PORT           本地相应192.168.1.6的arp请求(修改包内容后原路返回包)

    table=21, priority=1,arp,dl_vlan=1,arp_tpa=192.168.1.9 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:fa:16:3e:97:bd:37,load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0xfa163e97bd37->NXM_NX_ARP_SHA[],load:0xc0a80109->NXM_OF_ARP_SPA[],IN_PORT           本地相应192.168.1.9的arp请求

    table=21, priority=1,arp,dl_vlan=1,arp_tpa=192.168.1.8                本地相应192.168.1.8的arp请求 actions=move:NXM_OF_ETH_SRC[]->NXM_OF_ETH_DST[],mod_dl_src:fa:16:3e:d5:8e:54,load:0x2->NXM_OF_ARP_OP[],move:NXM_NX_ARP_SHA[]->NXM_NX_ARP_THA[],move:NXM_OF_ARP_SPA[]->NXM_OF_ARP_TPA[],load:0xfa163ed58e54->NXM_NX_ARP_SHA[],load:0xc0a80108->NXM_OF_ARP_SPA[],IN_PORT

    table=21, priority=0 actions=resubmit(,22)                            走表22
    ===============================================================================================================================================================
    table=22, n_packets=12, n_bytes=1062, idle_age=259, hard_age=494, dl_vlan=1 actions=strip_vlan,set_tunnel:0x138f,output:3      去掉vlan设置tunid 0x138f,走3号出口

    table=22, priority=0 actions=drop                                      未匹配丢弃包
    ===============================================================================================================================================================




## 现在我们来揭示上面的配置错误！

    配置了local_ip="10.10.0.52"的网卡不应该绑定到br-tun上........
    因为你看br-tun里根本就没有其他port可以出去的规则(br-ex的流表里只有一个normal规则)
    vxlan这个port最后会找本地一个ip为10.10.0.52的网卡把包发出去
    这个错误是因为之前看到br-ex绑定物理网卡,所以我也以为br-tun也要绑物理网卡,导致我两台虚拟机不能互通
