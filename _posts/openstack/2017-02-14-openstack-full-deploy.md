---
layout: post
title:  "OpenStack Mitaka从零开始 布署一套可用openstack环境"
date:   2016-12-06 15:05:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}


## 基础OpenStack Mitaka部署配置过程

因为是要部署的不是一个存粹的测试环境、而是一套可用的私有云环境，同时为了更好的学习虚拟化网络,所以网络选择dvr和vxlan.当然openstack也选择了支持domain的版本(公有云中账号内可以创建项目并实现项目隔离)


---

#### 部署的开始前首先要确定

##### 需少多少台机器(不包括浏览器的访问端)


    最少：1台    1台最大的问题在于、无法测试你跨机器的网络是否已经通畅
    推荐：2台    2台已经可以测试分布在两台物理机上的同网络同子网的虚拟机是否可达
    最好：3台+   3台可以把openstack的 消息通信/数据库 独立出来,更方便理解openstack所需网络

---

#### 在说明网卡需求和交换机需求之前、我们先说明一下openstack的网络划分

首先,要把openstack用的网络和openstack里的虚拟机用的网络分开来

    虚拟机内部网络分为
    内网：就是公有云服务器上看到的内网ip所在的网络
          这部分在openstack中是先由用户创建网络(network),然后在网络(network)中创建子网(subnet)
          然后创建虚拟机的是后选择网络(network)后,生成的虚拟机将从dhcp中分配到已用IP最少的子网(subnet)的ip, 在openstack中,虚拟机的网络隔离是project/network



    外网：就是大部分云上看不到的外网地址,在控制台里购买并绑定给虚拟机实例的ip


---

##### 机器需要多少张网卡

    最少：1个


##### 需要多少个交换机（和网卡数量对应,如果交换机可以按端口隔离vlan的话,每个vlan算多一个交换机）


    最少：1台     所有网段都混在一个交换机里,除非你已经很熟悉openstack的网络(如果你很熟悉也基本不用看这篇文了)
    推荐: 3台     openstack的 消息通信/数据库的走一台    openstack数据网络一台   openstack外部网络一台
    最好: 4台     管理网络单独加一台(这种情况下访问openstack的浏览器也在这个网络上)
    网络具体说明在下面机器需要多少个网卡的部分说明





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

    计算节点部分
    nova-compute
    neutron-openvswitch-agent
    neutron-l3-agent
    neutron-metadata-agent
    nova-novncproxy
    neutron-dhcp-agent.service

    因为cinder是非必要部件、布署cinder需要较大的磁盘空间
    需要学习Ceph分布式文件系统,所以不并未使用
