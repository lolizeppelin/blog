---
layout: post
title:  "vxlan中的vlan id是如何设置的"
date:   2017-02-23 15:05:00 +0800
categories: "网络基础"
tag: ["openstack", "linux"]
---

* content
{:toc}


通常vxlan的简介就在继续贴了,这里主要解决看vxlan的如下两个疑问
> 1. 有了vxlan id还要vlan id干什么用
> 2. vxlan中的vlan id在openstack里哪里配置, 交换机上要如何设置端口vlan?


解决问题之前,贴下vxlan的帧结构还是必要的

![](/styles/images/vxlan_with_vlan.png)


首先,openstack里没有和vxlan所用vlan id相关的配置文件。因为这个原因我一直也没搞明白这个vlan id的作用，后来因为要解决其他问题就把这个问题搁置了,直到昨天有人问我这么一个问题

    openstack的租户隔离不是有VLAN、GRE和VXLAN么，
    但是无论哪种在内部br-int都有个内部vlan tag，
    这个vlan tag是随机分配的吗？

带着疑惑看了下ovs的流表

     cookie=xxx,...,tun_id=0x138f actions=mod_vlan_vid:9,resubmit(,9）

也就是说,vlan的id是根据vxlan的值来设置的,那TM就产生了两个问题
>1. 原来的vlan id是什么
>2. 新的vlan id又是根据什么来生成的

为了解决这个问题,我分别看了两台机器上的流表

    机器A cookie=xxx,...,tun_id=0x138f actions=mod_vlan_vid:9,resubmit(,9)
    机器B cookie=xxx,...,tun_id=0x138f actions=mod_vlan_vid:7,resubmit(,9)

A机和B机器对同一个vxlan id(tun_id)打上的vlan id不一样！难道是TM随机的？

我们再看一条出去的规则

    cookie=xxx...,dl_vlan=9,dl_dst=fa:16:3e:67:cb:a7 actions=strip_vlan,mod_dl_src:fa:16:3e:29:54:8c,output:35

我操,打上的vlan标记又去除了？那出去的vxlan包不是没vlan标记了？为了验证这点...我们在物理出口上抓包看

![](/styles/images/vxlan_without_vlan.png)

卧槽....看见了么,出去的vxlan包根本没有vlan标记...是0x0800

# 也就是说所有文章介绍vxlan的时候都给vxlan包加上vlan标签是个很大的误导

---

现在我们来总结下

    vxlan包就是一个普通的udp包后面跟上vxlan包头
    vxlan包和vlan标签没有必然关系！！
    vxlan包可以有vlan标记也可以没有vlan标记
    openstack的vxlan从物理网卡口出去的是后是没有vlan标记的
    也就是说br-tun用的网卡要接在交换机的access口上
    也就是说交换机上配什么vlan是和vxlan的vlan id没有任何关系

那么现在回到了最初的问题1，有了vxlan id还要vlan id干什么用。为什么ovs在流表里给vxlan包打上vlan标记,出去的时候又剥离掉是在干什么。

当然是TM在做隔离啊.............vxlan、vlan不都是用来隔离的么

#### 那为什么vxlan要用vlan来隔离？

上面的问题其实就是答案....回想下这句话

    ovs在流表里给vxlan包打上vlan标记,出去的时候又剥离掉

vxlan打上的vlan标记是为了在"当前节点"内做隔离而存在的

#### 因为openvswitch中根本没有通过vxlan来隔离的代码！！vxlan的隔离不是原生的

自己设想看看,vlan协议扩展下字段就可以实现vxlan的功能,为什么不把vxlan放在以太网层而在udp层的下方

为的就是一个目的

## 兼容

如果假设我们直接在以太网层实现vxlan协议,那么老的设备都不能用了,同样,虚拟设备也要加入通过vxlan的id来隔离的代码

于是,为了不影响旧环境,出现了这样的vxlan实现方式

    ovs里识别vxlan包头,然后给vxlan包打上vlan标记(vxlan id和vlan id一一对应)
    这样的话ovs内部代码根本就不用修改就实现了vxlan隔离(所有的sdn软件肯定都能处理vlan)
    然后vxlan包出去的时候再把vlan标记剥离(物理网卡发出的vxlan包没有vlan标记)
    出去以后在物理网络上的包只当作一个普通的udp包发往支持vxlan的设备,所以自然也不用vlan标记
    交换机设置什么vlan和vxlan包么有任何关系

    只要节点内部vlan id不冲突,流表可用用任意vlan id
    因为这些vlan id只用于节点内部隔离,出去的时候也会剥离
    所以各个节点vxlan上的vlan id没有任何关系


sdn中这种非原生的vxlan实现方式最大的优点就是不用大量修改代码,只要在入口出口识别vxlan包就可以了

当然也有缺点,一个节点中最多只能支持和vlan id数量相同的vxlan,但目前来看,一个节点内肯定是没有那么多需要隔离的vxlan id存在的

一段时间后openvwitch中可能会原生的处理vxlan包,那么就不需要根据vxlan id设置vlan然后再剥离的操作了

看到这里,上面所有的疑问都可以解决了。

至于vxlan和vlan网络混用,那是要支持vxlan的设备做网关的事情,就不是这里的讨论范围了
