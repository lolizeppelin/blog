---
layout: post
title:  "网卡/bond网卡自动并绑定到ovs bridge"
date:   2014-08-16 15:05:00 +0800
categories: "虚拟化"
tag: ["linux"]
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
