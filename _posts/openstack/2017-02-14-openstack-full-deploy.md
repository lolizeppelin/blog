---
layout: post
title:  "OpenStack Mitaka从零开始 布署一套可用openstack环境"
date:   2017-02-14 15:05:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}


## 基础OpenStack Mitaka部署配置过程

因为是要部署的不是一个存粹的测试环境、而是一套可用的私有云环境，同时为了更好的学习虚拟化网络,所以网络选择ovs、dvr和vxlan.当然openstack也选择了支持domain的版本(类似公有云中账号内可以创建项目并实现项目隔离)

---

#### 部署的开始前首先要确定下到底需要点什么硬件

##### 需少多少台机器(不包括浏览器的访问端)

    最少：1台    1台最大的问题在于、无法测试你跨机器的网络是否已经通畅
    推荐：2台    2台已经可以测试分布在两台物理机上的同网络同子网的虚拟机是否可达
    最好：3台+   3台可以把openstack的 消息通信/数据库 独立出来,更方便理解openstack所需网络

---

#### 在说明网卡需求和交换机需求之前、我们先说明一下openstack的网络划分

首先,要把openstack用的网络和openstack里的虚拟机自身网络分开来

##### 虚拟机自身网络分为

    内网：就是公有云虚拟机看到的内网ip所在的网络
    外网：就是大部分云上虚拟机看不到的外网地址,在控制台里购买并绑定给虚拟机实例的ip所在网络

    虚拟机网络IP是在openstack中创建的,这里不得不提及openstack中为了管理虚拟机网络所用的几个概念

    网络(network), 由domain中的user创建
    子网(subnet),  由domain中的user在network下创建，可以在同一个network下创建多个子网
                   这里的subnet是用于管理我们通常意义上的子网的一个对象
                   subnet可以设置网段(例如192.168.1.0)、可分配的网络范围(192.168.1.2-192.168.1.10)
                   对应的网关IP,dhs服务器地址等,更接近dhcp的设置
                   一个network中可以有多个subnet
    路由(route),   由domain中的user创建,一般用于让内网IP转发到外网IP上

    在同一个网络(network)中就相当于在同一个交换机中,这个交换机中可以有多个网段不同的子网在上面跑
    不同的subnet互通需要通过route

    有点歧义的是,虚拟机创建的是后选择网络的时候是直接选择network,并不能选择network再选择subnet
    所以,当所选的network下有多个subnet的时候,虚拟机实例究竟用的是哪个subnet就得看代码了,这部分待续

    内网和外网都是通过相同的方式创建,不同之处在于
    外网的网络(network)需要由cloud admin(openstack的最高管理员)创建,创建的时候勾选外部网络、共享
    然后再创建外部网络所能使用的子网并定义IP段(公网IP地址池)、这里的网关就是外部的路由器了

    虚拟机创建的时候一般都只选择由自己创建的network,也就是只有内网IP
    需要外网IP的虚拟机需要通过创建浮动IP，然后绑定到对应虚拟机上
    具体实现比较复杂,等到对openstack比较熟悉以后
    可以参考http://www.lolizeppelin.com/2017/02/05/neutron-wan-route/
    我们暂时理解为虚拟机只有内网

##### openstack所用的网络分为(下面所说的外网不一定真要外网地址、交换机上vlan划分一个和办公网络平行的网络就可以了)

    外部网络: 这部分用于给内部虚拟机能路由到外部网络、外部网络访问到颞部虚拟机
             这个网络对需要连接到外网，但是不一定有IP地址
             对应到虚拟机网桥就是br-ex,一般要对应一个独立网卡,

    数据网络: 这部分网络最复杂,在熟悉之前简单理解为br-tun的网络出口
             一般也要对应一个独立网卡,一定有一个IP地址

    内部网络：这部分就是openstack通信用的网络,用于链接消息通信服务器、openstack数据库
             各个openstack组件间相互通信，建议使用独立网卡,当然有IP地址

    管理网络：这个网络就是相当于登陆管理openstack的的网络,需要能联通外网
             也建议独立网卡

    我们keyston创建endpoint有三种internal、public、admin
    public对应的就是就是管理网络,需要外网可以访问(浏览器需要和管理网络通信)
    internal对应的就是内部网络,这部分的endpoint是openstack各个组件通信时走的网络
    admin其实也是内部网络,其实和internal是一样的,在internal url的不同端口上而已
    本文用到的组件也只有keystone分为public/internal/admin(和internal的endpoint端口不同)
    其他组件的internal/admin是同一个地址、或者说根本没有用到internal地址

    如果没有那么多网卡的话,一般在在br-ex上配置ip,让br-ex兼职管理网络

---

##### 看完上面我们,机器需要多少张网卡就比较明确了

    最少：1张  除非你非常熟悉openstack网络,否则不要只用一张网卡
    实在没办法： 2张   数据网络、内部网络用同一张网卡(ip建议用2个便于理解)
                      管理网络和外部网络共用一张网卡(一个ip)
    推举： 3张        数据网络1张、内部网络一张、管理网络和外部网络共用一张网卡(一个ip)
    推荐： 4张        每个网络一张网卡
    顺便说下,1000M的双口intel网卡100块,100M的更便宜,这点钱都愿花就别学了

##### 需要多少个交换机（和网卡数量对应,如果交换机可以按端口隔离vlan的话,每个vlan算多一个交换机）

    最少：1台     所有网段都混在一个交换机里,除非你已经很熟悉openstack的网络(如果你很熟悉也基本不用看这篇文了)
    推荐: 3台     openstack的 消息通信/数据库的走一台    openstack数据网络一台   openstack外部网络一台
    最好: 4台     管理网络单独加一台(这种情况下访问openstack的浏览器也在这个网络上)


硬件的最后写下推荐的服务器

    二手 HP DL 360 g7 大概2000左右有4个网卡
    在随便来个支持vlan和三层转发的交换机

---

### 硬件准备完毕我们及可以开始配置了

一套基础的openstack包含以下组件

    控制节点部分
    keystone
    nova-api(os-compute-api和metadata-api)
    nova-conductor
    nova-scheduler
    nova-consoleauth
    glance-api
    glance-registry
    neutron-server

    计算/网络节点部分
    nova-compute
    neutron-openvswitch-agent
    neutron-l3-agent
    neutron-metadata-agent
    nova-novncproxy
    neutron-dhcp-agent.service

    因为cinder是非必要部件、布署cinder需要较大的磁盘空间
    需要学习Ceph分布式文件系统,所以我们暂时不接触这个组件

    因为使用dvr我们每个计算节点都是网络节点,单独的网络节点不是必要的
    单独的网络节点一般专门跑dvr-snat


待续
