---
layout: post
title:  "OpenStack Mitaka从零开始 nova通过neutron分配网络的过程(1)"
date:   2016-12-14 15:05:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}


nova创建虚拟机的时候最终会调用allocate_for_instance去创建网络,

过程为

    ComputeManager实例中
    _build_and_run_instance---_do_build_and_run_instance---
    _build_and_run_instance---_build_resources---_build_networks_for_instance
    _allocate_network---_allocate_network_async---network_api.allocate_for_instance

    network_aipj就是neutronv2.api.API
    __
---

我们通过neutronv2.api.API来分析下neutron创建网络的过程

---
    这里是allocate_for_instance


```python
def allocate_for_instance(self, context, instance, **kwargs):
    """虚拟器实例获取网络资源
    :param context: The request context.
    :param instance: nova.objects.instance.Instance object.
    :param requested_networks: optional value containing
        network_id, fixed_ip, and port_id
        这个参数表示用户指定了网络设置,不传进来将由neutron分配
    :param security_groups: security groups to allocate for instance
    :param macs: 这个参数和requested_networks类似,用于指定虚拟机网卡的mac,
                 libvirt的驱动不会自己设置mac,由neutron分配
    :param dhcp_options: 指定启动的dpcp信息,livirt驱动也设置为空
    :param bind_host_id: the host ID to attach to the ports being created.
    """
    hypervisor_macs = kwargs.get('macs', None)
    # 这里就是和neutron通信的类,M版用的是neutronclient.v2_0
    neutron = get_client(context)
    # 转成admin的api接口
    port_client = (neutron if not
                   self._has_port_binding_extension(context,
                       refresh_cache=True, neutron=neutron) else
                   get_client(context, admin=True))
    # Store the admin client - this is used later
    admin_client = port_client if neutron != port_client else None
    LOG.debug('allocate_for_instance()', instance=instance)
    if not instance.project_id:
        msg = _('empty project id for instance %s')
        raise exception.InvalidInput(
            reason=msg % instance.uuid)
    requested_networks = kwargs.get('requested_networks')
    dhcp_opts = kwargs.get('dhcp_options', None)
    bind_host_id = kwargs.get('bind_host_id')
    # 看下面_process_requested_networks部分
    # 当没有指定网络的时候(requested_network为空)
    # 几个返回值分别是空列表和空字典
    # ordered_networks表示已经指定好(经过校验的)的网络实例列表
    ports, net_ids, ordered_networks, available_macs = (
        self._process_requested_networks(context,
            instance, neutron, requested_networks, hypervisor_macs))
    # 通过neutronclient,也就是请求neutron的server
    # 获取当前所有准备好并符合条件的网络列表
    nets = self._get_available_networks(context, instance.project_id,
                                        net_ids, neutron=neutron)
    # 可用网络列表为空
    if not nets:
        # 如果用户指定了网络
        if requested_networks:
            for request in requested_networks:
                # 这里表示指定了网络,但是没有在neutron分配好网络接口(port)
                if not request.port_id and request.network_id:
                    raise exception.NetworkNotFound(
                        network_id=request.network_id)
        else:
            # 直接返回空网络列表
            LOG.debug("No network configured", instance=instance)
            return network_model.NetworkInfo([])

    # 没有指定网络, 或者指定了单网络模式
    if (not requested_networks
        or requested_networks.is_single_unspecified):
        # 没有指定网络或单网络模式找不到任何网络
        if not nets:
            raise exception.InterfaceAttachFailedNoNetwork(
                project_id=instance.project_id)
        # 单网络模式发现网络数量大于1, 报错
        if len(nets) > 1:
            msg = _("Multiple possible networks found, use a Network "
                     "ID to be more specific.")
            raise exception.NetworkAmbiguous(msg)
        # 自动选择第一个网络,看样子不指定网络的话总会选择返回的第一个
        # ordered_networks中至少有一个值,就是返回的网络列表中的第一个
        ordered_networks.append(
            objects.NetworkRequest(network_id=nets[0]['id']))

    # 这个代码内部实现比较复杂,看名字应该检查外部网络授权的
    # 如果当前context未授权,要检查网络列表(nets)中的网络是否有share标记
    # 列表中只要有网络不存在shere标记就报错ExternalNetworkAttachForbidden
    self._check_external_network_attach(context, nets)

    security_groups = kwargs.get('security_groups', [])
    security_group_ids = self._process_security_groups(
                                instance, neutron, security_groups)

    preexisting_port_ids = []
    created_port_ids = []
    ports_in_requested_order = []
    nets_in_requested_order = []
    for request in ordered_networks:
        # Network lookup for available network_id
        network = None
        for net in nets:
            if net['id'] == request.network_id:
                network = net
                break
        # if network_id did not pass validate_networks() and not available
        # here then skip it safely not continuing with a None Network
        else:
            continue

        nets_in_requested_order.append(network)
        # If security groups are requested on an instance then the
        # network must has a subnet associated with it. Some plugins
        # implement the port-security extension which requires
        # 'port_security_enabled' to be True for security groups.
        # That is why True is returned if 'port_security_enabled'
        # is not found.
        if (security_groups and not (
                network['subnets']
                and network.get('port_security_enabled', True))):

            raise exception.SecurityGroupCannotBeApplied()
        zone = 'compute:%s' % instance.availability_zone
        port_req_body = {'port': {'device_id': instance.uuid,
                                  'device_owner': zone}}
        try:
            self._populate_neutron_extension_values(
                context, instance, request.pci_request_id, port_req_body,
                network=network, neutron=neutron,
                bind_host_id=bind_host_id)
            if request.port_id:
                port = ports[request.port_id]
                port_client.update_port(port['id'], port_req_body)
                preexisting_port_ids.append(port['id'])
                ports_in_requested_order.append(port['id'])
            else:
                created_port = self._create_port(
                        port_client, instance, request.network_id,
                        port_req_body, request.address,
                        security_group_ids, available_macs, dhcp_opts)
                created_port_ids.append(created_port)
                ports_in_requested_order.append(created_port)
            self._update_port_dns_name(context, instance, network,
                                       ports_in_requested_order[-1],
                                       neutron)
        except Exception:
            with excutils.save_and_reraise_exception():
                self._unbind_ports(context,
                                   preexisting_port_ids,
                                   neutron, port_client)
                self._delete_ports(neutron, instance, created_port_ids)
    nw_info = self.get_instance_nw_info(
        context, instance, networks=nets_in_requested_order,
        port_ids=ports_in_requested_order,
        admin_client=admin_client,
        preexisting_port_ids=preexisting_port_ids,
        update_cells=True)
    # NOTE(danms): Only return info about ports we created in this run.
    # In the initial allocation case, this will be everything we created,
    # and in later runs will only be what was created that time. Thus,
    # this only affects the attach case, not the original use for this
    # method.
    return network_model.NetworkInfo([vif for vif in nw_info
                                      if vif['id'] in created_port_ids +
                                      preexisting_port_ids])

```



---
    下面是_process_requested_networks


```python
def _process_requested_networks(self, context, instance, neutron,
                                requested_networks, hypervisor_macs=None):
    """Processes and validates requested networks for allocation.

    Iterates over the list of NetworkRequest objects, validating the
    request and building sets of ports, networks and MAC addresses to
    use for allocating ports for the instance.

    :param instance: allocate networks on this instance
    :type instance: nova.objects.Instance
    :param neutron: neutron client session
    :type neutron: neutronclient.v2_0.client.Client
    :param requested_networks: list of NetworkRequests
    :type requested_networks: nova.objects.NetworkRequestList
    :param hypervisor_macs: None or a set of MAC addresses that the
        instance should use. hypervisor_macs are supplied by the hypervisor
        driver (contrast with requested_networks which is user supplied).
        NB: NeutronV2 currently assigns hypervisor supplied MAC addresses
        to arbitrary networks, which requires openflow switches to
        function correctly if more than one network is being used with
        the bare metal hypervisor (which is the only one known to limit
        MAC addresses).
    :type hypervisor_macs: set
    :returns: tuple of:
        - ports: dict mapping of port id to port dict
        - net_ids: list of requested network ids
        - ordered_networks: list of nova.objects.NetworkRequest objects
            for requested networks (either via explicit network request
            or the network for an explicit port request)
        - available_macs: set of available MAC addresses to use if creating
            a port later; this is the set of hypervisor_macs after removing
            any MAC addresses from explicitly requested ports.
    :raises nova.exception.PortNotFound: If a requested port is not found
        in Neutron.
    :raises nova.exception.PortNotUsable: If a requested port is not owned
        by the same tenant that the instance is created under. This error
        can also be raised if hypervisor_macs is not None and a requested
        port's MAC address is not in that set.
    :raises nova.exception.PortInUse: If a requested port is already
        attached to another instance.
    :raises nova.exception.PortNotUsableDNS: If a requested port has a
        value assigned to its dns_name attribute.
    """

    available_macs = None
    if hypervisor_macs is not None:
        # Make a copy we can mutate: records macs that have not been used
        # to create a port on a network. If we find a mac with a
        # pre-allocated port we also remove it from this set.
        available_macs = set(hypervisor_macs)

    ports = {}
    net_ids = []
    ordered_networks = []
    if requested_networks:
        for request in requested_networks:

            # Process a request to use a pre-existing neutron port.
            if request.port_id:
                # Make sure the port exists.
                port = self._show_port(context, request.port_id,
                                       neutron_client=neutron)
                # Make sure the instance has access to the port.
                if port['tenant_id'] != instance.project_id:
                    raise exception.PortNotUsable(port_id=request.port_id,
                                                  instance=instance.uuid)

                # Make sure the port isn't already attached to another
                # instance.
                if port.get('device_id'):
                    raise exception.PortInUse(port_id=request.port_id)

                # Make sure that if the user assigned a value to the port's
                # dns_name attribute, it is equal to the instance's
                # hostname
                if port.get('dns_name'):
                    if port['dns_name'] != instance.hostname:
                        raise exception.PortNotUsableDNS(
                            port_id=request.port_id,
                            instance=instance.uuid, value=port['dns_name'],
                            hostname=instance.hostname)

                # Make sure the port is usable
                if (port.get('binding:vif_type') ==
                    network_model.VIF_TYPE_BINDING_FAILED):
                    raise exception.PortBindingFailed(
                        port_id=request.port_id)

                if hypervisor_macs is not None:
                    if port['mac_address'] not in hypervisor_macs:
                        LOG.debug("Port %(port)s mac address %(mac)s is "
                                  "not in the set of hypervisor macs: "
                                  "%(hyper_macs)s",
                                  {'port': request.port_id,
                                   'mac': port['mac_address'],
                                   'hyper_macs': hypervisor_macs},
                                  instance=instance)
                        raise exception.PortNotUsable(
                            port_id=request.port_id,
                            instance=instance.uuid)
                    # Don't try to use this MAC if we need to create a
                    # port on the fly later. Identical MACs may be
                    # configured by users into multiple ports so we
                    # discard rather than popping.
                    available_macs.discard(port['mac_address'])

                # If requesting a specific port, automatically process
                # the network for that port as if it were explicitly
                # requested.
                request.network_id = port['network_id']
                ports[request.port_id] = port

            # Process a request to use a specific neutron network.
            if request.network_id:
                net_ids.append(request.network_id)
                ordered_networks.append(request)

    return ports, net_ids, ordered_networks, available_macs
```
