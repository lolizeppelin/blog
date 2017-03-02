---
layout: post
title:  "OpenStack Mitaka从零开始 nova通过neutron分配网络的过程(n)"
date:   2016-12-20 15:05:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}


neutron ml(openvswitch) agent初始化并创建br的过程

    文件neutron.plugins.ml2.drivers.openvswitch.agent.ovs_neutron_agent.py

---

    agent守护进程的init

```python
class OVSNeutronAgent(sg_rpc.SecurityGroupAgentRpcCallbackMixin,
                      l2population_rpc.L2populationRpcCallBackTunnelMixin,
                      dvr_rpc.DVRAgentRpcCallbackMixin):
    target = oslo_messaging.Target(version='1.4')

    def __init__(self, bridge_classes, conf=None):

        super(OVSNeutronAgent, self).__init__()
        self.conf = conf or cfg.CONF
        self.ovs = ovs_lib.BaseOVS()
        agent_conf = self.conf.AGENT
        ovs_conf = self.conf.OVS

        # ....略过部分代码
        # 通过tunnel_types设置判断当天agent是否启动tun
        self.tunnel_types = agent_conf.tunnel_types or []
        if self.tunnel_types:
            self.enable_tunneling = True
        else:
            self.enable_tunneling = False
        # 初始化br_int, br_int类的实例不在setup_integration_br里生成
        # 目的应该是为了reload的时候self.int_br不重新初始化
        self.int_br = self.br_int_cls(ovs_conf.integration_bridge)
        self.setup_integration_br()
        # 初始化physical br
        self.bridge_mappings = self._parse_bridge_mappings(
            ovs_conf.bridge_mappings)
        self.setup_physical_bridges(self.bridge_mappings)
        # 初始化br_tun
        if self.enable_tunneling:
            self.setup_tunnel_br(ovs_conf.tunnel_bridge)
        # ....略过其余代码
```




---

    下面是是br_int的初始化

```python
def setup_integration_br(self):
    '''Setup the integration bridge.

    '''
    # Ensure the integration bridge is created.
    # ovs_lib.OVSBridge.create() will run
    #   ovs-vsctl -- --may-exist add-br BRIDGE_NAME
    # which does nothing if bridge already exists.
    # br_int是完全由neutron-agent管理的
    self.int_br.create()
    self.int_br.set_secure_mode()
    self.int_br.setup_controllers(self.conf)

    if self.conf.AGENT.drop_flows_on_start:
        # Delete the patch port between br-int and br-tun if we're deleting
        # the flows on br-int, so that traffic doesn't get flooded over
        # while flows are missing.
        self.int_br.delete_port(self.conf.OVS.int_peer_patch_port)
        self.int_br.delete_flows()
    self.int_br.setup_default_table()

```

---

    下面是br_tun的初始化

```python
def setup_tunnel_br(self, tun_br_name=None):
    '''(re)initialize the tunnel bridge.

    Creates tunnel bridge, and links it to the integration bridge
    using a patch port.

    :param tun_br_name: the name of the tunnel bridge.
    '''
    if not self.tun_br:
        self.tun_br = self.br_tun_cls(tun_br_name)
    # tun_br.create() won't recreate bridge if it exists, but will handle
    # cases where something like datapath_type has changed
    # 注意看上面注释,在tun_br存在的情况下不会创建,只修改datapath_type
    # 就目前代码来看,tun_br没有外链到具体物理网卡的过程
    # 所以,应该是在系统层面设置好br_tun和并链接到物理网卡
    # neutron只会重新设置br_tun的datapath_type
    # (注意,当时的上面关于br_tun的理解是错的,br_tun和br_ex不一样,不要在系统层设置好br-tun)
    # 这里的处理方式和setup_physical_bridges是一样的
    # 都是在neutron外部管理br和物理网卡的关联
    # 只有br_int是完全由neutron agent管理的
    self.tun_br.create(secure_mode=True)
    self.tun_br.setup_controllers(self.conf)
    # 在br_int和br_tun上创建br_int和bt_tun相连的端口
    # 做法是通过ovsdb分别在br_int和br_tun上设置local port和peer remote port
    if (not self.int_br.port_exists(self.conf.OVS.int_peer_patch_port) or
            self.patch_tun_ofport == ovs_lib.INVALID_OFPORT):
        self.patch_tun_ofport = self.int_br.add_patch_port(
            self.conf.OVS.int_peer_patch_port,
            self.conf.OVS.tun_peer_patch_port)
    if (not self.tun_br.port_exists(self.conf.OVS.tun_peer_patch_port) or
            self.patch_int_ofport == ovs_lib.INVALID_OFPORT):
        self.patch_int_ofport = self.tun_br.add_patch_port(
            self.conf.OVS.tun_peer_patch_port,
            self.conf.OVS.int_peer_patch_port)
    if ovs_lib.INVALID_OFPORT in (self.patch_tun_ofport,
                                  self.patch_int_ofport):
        LOG.error(_LE("Failed to create OVS patch port. Cannot have "
                      "tunneling enabled on this agent, since this "
                      "version of OVS does not support tunnels or patch "
                      "ports. Agent terminated!"))
        sys.exit(1)
    if self.conf.AGENT.drop_flows_on_start:
        self.tun_br.delete_flows()
```


---

    下面是root的启动函数

```python
def daemon_loop(self):
    # Start everything.
    LOG.info(_LI("Agent initialized successfully, now running... "))
    signal.signal(signal.SIGTERM, self._handle_sigterm)
    if hasattr(signal, 'SIGHUP'):
        signal.signal(signal.SIGHUP, self._handle_sighup)
    with polling.get_polling_manager(
        self.minimize_polling,
        self.ovsdb_monitor_respawn_interval) as pm:

        self.rpc_loop(polling_manager=pm)
```
