---
layout: post
title:  "网卡/bond网卡自动并绑定到ovs bridge"
date:   2014-08-16 15:05:00 +0800
categories: "虚拟化"
tag: ["linux", "openvwitch"]
---

* content
{:toc}


---
单网卡自动绑定到ovs bridge

ifcfg-br-tun

    DEVICE=br-tun
    ONBOOT=yes
    DEVICETYPE=ovs
    TYPE=OVSBridge
    BOOTPROTO=static
    IPADDR=192.168.2.202
    NETMASK=255.255.255.0
    HOTPLUG=no

ifcfg-eth0

    DEVICE=eth0
    ONBOOT=yes
    NM_CONTROLLED=no
    BOOTPROTO=none
    DEVICETYPE=ovs
    TYPE=OVSPort
    OVS_BRIDGE=br-tun


---
网卡bond并自动绑定到ovs bridge


ifcfg-br_lan

    DEVICE=br_lan
    ONBOOT=yes
    DEVICETYPE=ovs
    TYPE=OVSBridge
    BOOTPROTO=static
    IPADDR=192.168.2.202
    NETMASK=255.255.255.0
    HOTPLUG=no


ifcfg-bond1

    DEVICE="bond1"
    BOOTPROTO=none
    NM_CONTROLLED=no
    ONBOOT=yes
    DEVICETYPE=ovs
    TYPE=OVSBond
    # 绑定到指定bridge
    OVS_BRIDGE=br_lan
    BOND_IFACES="eth2 eth3"
    OVS_OPTIONS="bond_mode=balance-slb lacp=off"


ifcfg-eth3

    DEVICE=eth3
    ONBOOT=yes
    NM_CONTROLLED=no
    BOOTPROTO=none


ifcfg-eth2

    DEVICE=eth3
    ONBOOT=yes
    NM_CONTROLLED=no
    BOOTPROTO=none


多ip配置

    DEVICE=br_lan:0
    IPADDR="192.168.0.4"
    NETMASK="255.255.255.0"
    ONBOOT="yes"
    BOOTPROTO="static"



配置好后，不需要使用ovs-vsctl命令，直接用/etc/init.d/network restart就弄OK了.实际命令都在

    /etc/sysconfig/network-scripts/ifup-ovs  
    /etc/sysconfig/network-scripts/

ifdown-ovs中运行了

    ovs-vsctl -t ${TIMEOUT} -- --fake-iface add-bond "$OVS_BRIDGE" "$DEVICE" ${BOND_IFACES} $OVS_OPTIONS ${OVS_EXTRA+-- $OVS_EXTRA}

如果使用

    OVS_OPTIONS="bond_mode=balance-tcp lacp=off"

    那么你的上层需要有负载均衡设备之类的直接丢对应的tcp ip包到bonding对应的网卡
    如果你直接ping会发现balance-tcp模式的bonding网卡的不响应arp..
    这个问题折磨我个把小时，搞得我一直以为哪里有问题网络没通。
    后来我直接换成active-backup就可以了，再试了下slb也能用就直接用slb模式了，真是卧槽。


具体的type类型参考/etc/sysconfig/network-scripts/ifup-ovs

```shell
case "$TYPE" in
	OVSBridge|OVSUserBridge)
		# If bridge already exists and is up, it has been configured through
		# other cases like OVSPort, OVSIntPort and OVSBond. If it is down or
		# it does not exist, create it. It is possible for a bridge to exist
		# because it remained in the OVSDB for some reason, but it won't be up.
		if [ "${TYPE}" = "OVSUserBridge" ]; then
			DATAPATH="netdev"
		fi
		if check_device_down "${DEVICE}"; then
			ovs-vsctl -t ${TIMEOUT} -- --may-exist add-br "$DEVICE" $OVS_OPTIONS \
			${OVS_EXTRA+-- $OVS_EXTRA} \
			${STP+-- set bridge "$DEVICE" stp_enable="${STP}"} \
			${DATAPATH+-- set bridge "$DEVICE" datapath_type="$DATAPATH"}
		else
			OVSBRIDGECONFIGURED="yes"
		fi

		# If MACADDR is provided in the interface configuration file,
		# we need to set it using ovs-vsctl; setting it with the "ip"
		# command in ifup-eth does not make the change persistent.
		if [ -n "$MACADDR" ]; then
			ovs-vsctl -t ${TIMEOUT} -- set bridge "$DEVICE" \
				other-config:hwaddr="$MACADDR"
		fi

		# When dhcp is enabled, the assumption is that there will be a port to
		# attach (otherwise, we can't reach out for dhcp). So, we do not
		# configure the bridge through rhel's ifup infrastructure unless
		# it is being configured after the port has been configured.
		# The "OVSINTF" is set only after the port is configured.
		if [ "${OVSBOOTPROTO}" = "dhcp" ] && [ -n "${OVSINTF}" ]; then
			case " ${OVSDHCPINTERFACES} " in
				*" ${OVSINTF} "*)
					BOOTPROTO=dhcp ${OTHERSCRIPT} ${CONFIG}
				;;
			esac
		fi

		# When dhcp is not enabled, it is possible that someone may want
		# a standalone bridge (i.e it may not have any ports). Configure it.
		if [ "${OVSBOOTPROTO}" != "dhcp" ] && [ -z "${OVSINTF}" ] && \
			[ "${OVSBRIDGECONFIGURED}" != "yes" ]; then
			${OTHERSCRIPT} ${CONFIG}
		fi
		exit 0
		;;
	OVSPort)
		ifup_ovs_bridge
		${OTHERSCRIPT} ${CONFIG} ${2}
		# The port might be already in the database but not yet
		# in the datapath.  So, remove the stale interface first.
		ovs-vsctl -t ${TIMEOUT} \
			-- --if-exists del-port "$OVS_BRIDGE" "$DEVICE" \
			-- add-port "$OVS_BRIDGE" "$DEVICE" $OVS_OPTIONS ${OVS_EXTRA+-- $OVS_EXTRA}
		OVSINTF=${DEVICE} /sbin/ifup "$OVS_BRIDGE"
		;;
	OVSIntPort)
		ifup_ovs_bridge
		ovs-vsctl -t ${TIMEOUT} -- --may-exist add-port "$OVS_BRIDGE" "$DEVICE" $OVS_OPTIONS -- set Interface "$DEVICE" type=internal ${OVS_EXTRA+-- $OVS_EXTRA}
		if [ -n "${OVSDHCPINTERFACES}" ]; then
			for _iface in ${OVSDHCPINTERFACES}; do
				/sbin/ifup ${_iface}
			done
		fi
		BOOTPROTO="${OVSBOOTPROTO}" ${OTHERSCRIPT} ${CONFIG} ${2}
		;;
	OVSBond)
		ifup_ovs_bridge
		for _iface in $BOND_IFACES; do
			/sbin/ifup ${_iface}
		done
		ovs-vsctl -t ${TIMEOUT} -- --may-exist add-bond "$OVS_BRIDGE" "$DEVICE" ${BOND_IFACES} $OVS_OPTIONS ${OVS_EXTRA+-- $OVS_EXTRA}
		OVSINTF=${DEVICE} /sbin/ifup "$OVS_BRIDGE"
		;;
	OVSTunnel)
		ifup_ovs_bridge
		ovs-vsctl -t ${TIMEOUT} -- --may-exist add-port "$OVS_BRIDGE" "$DEVICE" $OVS_OPTIONS -- set Interface "$DEVICE" type=$OVS_TUNNEL_TYPE $OVS_TUNNEL_OPTIONS ${OVS_EXTRA+-- $OVS_EXTRA}
		;;
	OVSPatchPort)
		ifup_ovs_bridge
		ovs-vsctl -t ${TIMEOUT} -- --may-exist add-port "$OVS_BRIDGE" "$DEVICE" $OVS_OPTIONS -- set Interface "$DEVICE" type=patch options:peer="${OVS_PATCH_PEER}" ${OVS_EXTRA+-- $OVS_EXTRA}
		;;
	OVSDPDKPort)
		ifup_ovs_bridge
		ovs-vsctl -t ${TIMEOUT} -- --may-exist add-port "$OVS_BRIDGE" "$DEVICE" $OVS_OPTIONS -- set Interface "$DEVICE" type=dpdk ${OVS_EXTRA+-- $OVS_EXTRA}
		;;
	OVSDPDKRPort)
		ifup_ovs_bridge
		ovs-vsctl -t ${TIMEOUT} -- --may-exist add-port "$OVS_BRIDGE" "$DEVICE" $OVS_OPTIONS -- set Interface "$DEVICE" type=dpdkr ${OVS_EXTRA+-- $OVS_EXTRA}
		;;
	OVSDPDVhostPort)
		ifup_ovs_bridge
		ovs-vsctl -t ${TIMEOUT} -- --may-exist add-port "$OVS_BRIDGE" "$DEVICE" $OVS_OPTIONS -- set Interface "$DEVICE" type=dpdkvhost ${OVS_EXTRA+-- $OVS_EXTRA}
		;;
	OVSDPDKVhostUserPort)
		ifup_ovs_bridge
		ovs-vsctl -t ${TIMEOUT} -- --may-exist add-port "$OVS_BRIDGE" "$DEVICE" $OVS_OPTIONS -- set Interface "$DEVICE" type=dpdkvhostuser ${OVS_EXTRA+-- $OVS_EXTRA}
		;;
	*)
		echo $"Invalid OVS interface type $TYPE"
		exit 1
		;;
esac

```


[参考网站](http://openvswitch.org/pipermail/discuss/2011-October/005845.htm)


顺便把linux网卡bond的详细再补充下, [参考原文](http://lcpljc.blog.51cto.com/200989/1054878)

方法/步骤

```text

1.   创建bond接口，在接口配置文件的路径下/etc/sysconfig/network-scripts/
#vi /etc/sysconfig/network-scripts/ifcfg-bond0
DEVICE="bond0"
NAME="bond0"
#TYPE=BOND
#BONDING_MASTER=yes
BOOTPROTO=static
IPADDR=172.16.0.1
NETMASK=255.255.255.0
GATEWAY=172.16.0.253
DNS1=202.106.0.20               --根据本地上网情况更改
NM_CONTROLLED=no        —与NetwordManager相关，必须关闭，且此服务器必须关闭，否则起不到绑定作用
ONBOOT=yes
USERCTL=no
BONDING_OPTS="mode=1 miimon=100"    ——mode=1  冗余模式
```
---

```text
BONDING_OPTS参数解释
     此参数用于指定网卡绑定时的属性，以下是对常用参数进行的解释：
         miimon参数：指定网卡故障时的切换时间间隔以ms为单位。
         primary参数：指定默认的主网卡设备。
     mode参数：
          0－即round-robin,轮询模式，所绑定的网卡会针对访问以轮询算法进行简单的平分。
             需要交换机开启端口聚合,负载均衡兼容错

          1－即active-backup,高可用模式，运行时只使用一个网卡，其余网卡作为备份
             在负载不超过单块网卡带宽或压力时建议使用,有特殊参数miimon用于间隔检查
             普通主备模式,不需要交换机设置

          2－即load balancing,基于HASH算法的负载均衡模式
             网卡的分流按照xmit_hash_policy的TCP协议层设置来进行HASH计算分流
             使各种不同处理来源的访问都尽量在同一个网卡上进行处理。通过源地址和目标地址来选择网卡出口
             需要交换机端口聚合

          3－即fault-tolerance,广播模式，所有被绑定的网卡都将得到相同的数据
             好处是当有对端交换机失效，我们感觉不到任何downtime
             一般用于十分特殊的网络需求比如金融行业，需要对两个互相没有连接的交换机发送相同的数据。
             需要特别的交换机配置

          4－即lacp,也就是802.3ab负载均衡模式，要求交换机也支持802.3ab(lacp)模式
             理论上服务器及交换机都支持此模式时，网卡带宽最高可以翻倍(如从1Gbps翻到2Gbps)
             需要交换机端口开启lacp

          5－即balance-tlb,适配器输出负载均衡模式，输出的数据会通过所有被绑定的网卡输出
             接收数据时则只选定其中一块网卡。如果正在用于接收数据的网卡发生故障
             则由其他网卡接管，要求所用的网卡及网卡驱动可通过ethtool命令得到speed信息。
             bond成员使用各自的mac，而不是上面几种模式是使用bond0接口的mac。
             设备开始时会发送免费arp，以主端口eth1的mac为源
             当客户端收到这个arp时就会在arp缓存中记录下这个mac对的ip。
             而在这个模式下，服务器每个端口在ping操作时，会根据算法算出出口
             地址不断变化时他，这时会负载到不同端口
             不需要特别的交换机配置

          6－即adaptive load balancing,适配器输入/输出负载均衡模式
             在"模式5"的基础上，在接收数据的同时实现负载均衡
             除要求ethtool命令可得到speed信息外，还要求支持对网卡MAC地址的动态修改功能。
             不需要特别的交换机配置

      xmit_hash_policy参数(此参数对mode参数中的2、4模式有影响)：

      layer1－通过MAC地址进行HASH计算。

   计算公式：(MACsrc⊕MACdest)% Nslave
      layer3+4－通过TCP及UDP端口及其IP地址进行HASH计算。
    计算公式：((portsrc⊕portdest)⊕(IPsrc⊕IPdest)) % Nslave

注意：
       mode参数中的0、2、3、4模式要求交换机支持"ports group"功能并能进行相应的设置
       例如在Cisco中要将所连接的端口设为"trunk group"。
       在H3C中就是interface Bridge-Aggregation

选择绑定模式的建议
    如果系统流量不超过单个网卡的带宽，请不要选择使用mode 1之外的模式
    因为负载均衡需要对流量进行计算，这对系统性能会有所损耗。
    建议mode 5、mode 6只在交换机不支持"ports group"的情况下选用。
    如果交换机及网卡都确认支持802.3ab，则实现负载均衡时尽量使用mode 4以提高系统性能。
```

```text
2.   修改要加入bond的两个物理口的接口配置文件如下（修改eth0）：
# vi /etc/sysconfig/network-scripts/ifcfg-eth0
DEVICE="eth0"
BOOTPROTO=none
MASTER=bond0
SLAVE=yes
ONBOOT=yes
USERCTL=no
3.   修改要加入bond的两个物理口的接口配置文件如下（修改eth1）：
# vi /etc/sysconfig/network-scripts/ifcfg-eth1
DEVICE="eth1"
BOOTPROTO=none
MASTER=bond0
SLAVE=yes
ONBOOT=yes
USERCTL=no
```

补充一下, 一般情况下我们直接在交换机上配置静态lacp,然后在服务器上配置

```text
BONDING_OPTS="mode=4"
```

如果用静态端口聚合要配置mode=0,这个模式并不是很好,会因为包序列混乱导致丢包重传
