---
layout: post
title:  "H3C WX3010E无线交换机配置"
date:   2017-01-18 15:05:00 +0800
categories: "网络基础"
tag: ["3层交换机"]
---

* content
{:toc}

首先感谢这篇[blog](http://blog.sina.com.cn/s/blog_51f491d30102uyuz.html)

首先是说明书,百度一下可以到H3C官网下载到

    H3C WX3000 系列有线无线一体化交换机交换引擎.pdf
    H3C WX系列无线控制产品 Web网管配置指导-6W106-整本手册.pdf
    H3C WX3010E有线无线一体化交换机安装指导-APW100-整本手册.pdf
    H3C S5000PV2_EI-CMW520-R1106 版本说明书.pdf


交换机分为两个部分

1.一个是交换机部分,就是和不同交换机一样,设置对应到设备上的rj45口,这部分的系统说明书称之为交换引擎
2.无线网络控制器,这部分的系统主要用于控制fit ap


先看交换模块


    GigabitEthernet1/0/1 ~ GigabitEthernet1/0/8
    对应交换机上的 1 ~ 8 Rj45 端口

    GigabitEthernet1/0/9 ~ GigabitEthernet1/0/10
    对应交换机上的光纤口(一般来说,每两个光千口和一个Rj45端口公用部分资源,也就是说使用两个光纤口会需要关闭对应的Rj45端口,反之一样,所以9-10需要关闭)


    GigabitEthernet1/0/11
    GigabitEthernet1/0/12
    上述两个汇聚为Bridge-Aggregation1, 用于链接WLAN控制模块,这里就是和普通交换机区别的地方



再看控制模块

    GigabitEthernet1/0/1     
    GigabitEthernet1/0/2
    上述两个汇聚为Bridge-Aggregation1, 用于链接交换模块








进入到模板
wlan service-template 21
关闭当前模板
service-template disable




VLAN 86 带有dhcp,用于给每个ap分配地址




### 这个文章作废,当时理解有问题需要重新设置

交换机sw-b
3个vlan


vlan 20，跳跃点vlan
    route interface 172.20.0.4/23
    port 1

vlan 30, 用于虚拟机外部网络(br-ex)

    route interface 172.30.0.1/24
    port 2~8


vlan  4001, 用于虚拟机内部/管理网络网络(ampq网络)

    port 9~16

vlan  4002, 用于虚拟机内部网络(br-tun)

    port 17-24

设置路由 0.0.0.0 172.20.0.1

具体命令
    # 登如交换机 此时前缀是<sw-b>
    # 进入system 此时前缀是[sw-b]
    sys
    # 列出端口
    display port-security
    # 创建/进入vlan 30
    vlan 30
    # 绑定端口1~8
    port GigabitEthernet1/0/1 to GigabitEthernet1/0/8
    # 解绑端口1
    undo port GigabitEthernet1/0/1
    # 检查vlan 30配置
    display vlan 30
    # vlan 30 interface设置
    interface vlan 30
    # 设置vlan 30的ip
    ip add 172.30.0.1 255.255.255.0
    # 返回到sys
    quit
    # 设置vlan 20
    vlan 20
    # 设置vlan 20 接口
    interface vlan 20
    # 设置vlan 20 IP
    ip add 172.20.0.4 255.255.254.0
    # 返回到上级
    quit
    # 进入port 1
    interface GigabitEthernet1/0/1
    # 设置port为trunk
    port link-type trunk
    # 设置port允许的vlan id  
    port trunk permit vlan 16 20 30
    # 设置trunk port的 pvid
    port trunk pvid vlan 20
    # 查看路由（不加verbose只能看到激活的路由）
    display ip routing-table verbose
    # 查看端口, Port link-type显示端口类型
    display interface GigabitEthernet 1/0/1
    # 查看vlan 接口信息
    display interface Vlan-interface
    # 设置全局路由
    ip route-static 0.0.0.0 0.0.0.0 172.20.0.1

---

交换机sw-a  3个vlan

    vlan 20, 原有IP段

        route interface 172.20.0.2/23
        port 1~8
        port 1 trunck 直连sw-b port 1

    vlan  4001, 用于虚拟机内部/管理网络网络(ampq网络)

        port 9~16

    vlan  4002, 用于虚拟机内部网络(br-tun)

        port 17-24

静态路由设置
设置路由 0.0.0.0 172.20.0.1
设置路由 172.30.0.0 172.20.0.4

具体命令
    sys
    vlan 20
    port GigabitEthernet1/0/1 to GigabitEthernet1/0/8
    interface vlan 20
    ip add 172.20.0.2 255.255.254.0
    quit
    vlan 4001
    port GigabitEthernet1/0/9 to GigabitEthernet1/0/16
    quit
    vlan 4002
    port GigabitEthernet1/0/17 to GigabitEthernet1/0/24
    quit
    interface GigabitEthernet1/0/1
    port link-type trunk
    port trunk permit vlan 20 30 4001 4002
    port trunk pvid vlan 20
    quit
    ip route-static 172.30.0.0 24 172.20.0.4
    ip route-static 0.0.0.0 0.0.0.0 172.20.0.1


http://blog.sina.com.cn/s/blog_51f491d30102uyuz.html

http://www.h3c.com.cn/Service/Document_Center/Wlan/WX/WX5000/Configure/Operation_Manual/H3C_WX_CG-6W104/02/201208/751246_30005_0.htm
