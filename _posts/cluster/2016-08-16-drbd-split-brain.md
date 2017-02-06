---
layout: post
title:  "drbd脑裂的一般处理"
date:   2016-08-16 15:05:00 +0800
categories: "troubleshooting"
tag: ["drbd"]
---


* content
{:toc}


drbdadm说明

	-- backend-options
	   All options following the doubly hyphen are considered backend-options. These are passed through to the
	   backend command. I.e. to drbdsetup, drbdmeta or drbd-proxy-ctl.
       drbdadm是几个命令工具的合集, --表示后面的参数穿给他实际调用的命令

	--discard-my-data
	   Use this option to manually recover from a split-brain situation. In case you do not have any automatic
	   after-split-brain policies selected, the nodes refuse to connect. By passing this option you make this node
	   a sync target immediately after successful connect.

       这个参数用与脑裂手动修复,如果你没有启用自动脑裂处理策略,那么节点将拒绝connect
       通过这个参数,链接后立刻使得当前节点会被立刻同步(参数名字丢弃自己的数据,所以一般先要设置当前节点为secondary)

	connect

	   Sets up the network configuration of the resource's device. If the peer device is already configured, the
	   two DRBD devices will connect. If there are more than two host sections in the resource you need to use the
	   --peer option to select the peer you want to connect to.

       主动链接对面资源


    disconnect

	   Removes the network configuration from the resource. The device will then go into StandAlone state.	   

    attach

	   Attaches a local backing block device to the DRBD resource's device.
       drbd先connect后,才在dev中创建设备的文件,attach就是创建设备文件

    detach

	   Removes the backing storage device from a DRBD resource's device.

    up

	   Is a shortcut for attach and connect.

       先connect再attach,相当于connect和attach两个动作打包

    down

	   Is a shortcut for disconnect and detach.



---

需要被同步的节点执行

	drbdadm secondary mydata
	drbdadm -- --discard-my-data connect mydata


主节点执行

	drbdadm connect mydata

---

如果上面方法不行,需要从节点全部擦除重置数据

被同步节点执行
    # 关闭设备文件,在这之前当然需要unmount
	drbdadm detach mysql
    # dd擦除drbd的元数据,假设drbd的resource的设备为/dev/sdb1
	dd if=/dev/zero bs=1M count=100 of=/dev/sdb1
	drbdadm down mysql
    # 重建元数据
	drbdadm create-md mysql

在主节点上执行

	drbdadm connect mysql
