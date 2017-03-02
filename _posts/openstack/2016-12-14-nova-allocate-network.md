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

    network_aipj就是nova.network.neutronv2.api.API
    __
---

我们通过nova.network.neutronv2.api中的API类来分析下nova申请网络的过程

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
    # 这个是申请入口传进来的列表,应该就是页面上选取的网络传入
    requested_networks = kwargs.get('requested_networks')
    dhcp_opts = kwargs.get('dhcp_options', None)
    bind_host_id = kwargs.get('bind_host_id')
    # 看下面_process_requested_networks部分
    # 当没有指定网络的时候(requested_network为空)
    # 几个返回值分别是空列表和空字典
    # ordered_networks表示已经过校验(校验requested_networks生成)的的网络申请列表
    # 如果requested_networks中的每个对象都不包含port_id或network_id,ordered_networks为空,其他几个返回值也是空
    # net_ids就是网络申请列表中每个对象的的network_id属性组成的列表,也就是需要申请的网络ID组成的列表
    # ports和hypervisor_macs需要判断网络申请列表中是否有port相关信息才生成的,默认为空
    ports, net_ids, ordered_networks, available_macs = (
        self._process_requested_networks(context,
            instance, neutron, requested_networks, hypervisor_macs))
    # 如果net_ids不为空,请求neutron的server,获取所有id在net_ids的网络
    # 如果前面的net_ids为空,请求neutron的server获取当前租户所有的所有网络列表 + 所有share网络列表
    # 最后通过id排序的上述列表列表并返回
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
            # 直接返回空网络列表,也就是说没有指定网络又找不到可用网络的情况下,不分配网络
            LOG.debug("No network configured", instance=instance)
            return network_model.NetworkInfo([])

    # 没有指定网络, 或者指定了单网络模式
    # is_single_unspecified是网络请求中只有一个网络,requested_networks比较复杂
    if (not requested_networks
        or requested_networks.is_single_unspecified):
        # 没有指定网络或单网络模式找不到任何网络
        if not nets:
            # 报错,也就是说在租户建立的网络的情况下,不指定网络会报错或使用单网络模式会报错
            raise exception.InterfaceAttachFailedNoNetwork(
                project_id=instance.project_id)
        # 单网络模式但是返回的网络数量大于1, 报错
        if len(nets) > 1:
            msg = _("Multiple possible networks found, use a Network "
                     "ID to be more specific.")
            # 也就是说在租户建立的网络的情况下,不指定网络会报错或使用单网络模式会,但发现有多个network也会报错
            raise exception.NetworkAmbiguous(msg)
        # ordered_networks在不指定网络的时候为空,能走到这里肯定只有一个网络,这个唯一的网络插到申请网络列表中
        ordered_networks.append(
            objects.NetworkRequest(network_id=nets[0]['id']))

    # 这个代码内部实现比较复杂,看名字应该检查外部网络授权的
    # 如果当前context未授权,要检查网络列表(nets)中的网络是否有share标记
    # 列表中只要有网络不存在shere标记就报错ExternalNetworkAttachForbidden
    self._check_external_network_attach(context, nets)

    security_groups = kwargs.get('security_groups', [])
    security_group_ids = self._process_security_groups(
                                instance, neutron, security_groups)
    # 已经存在的port 列表,
    preexisting_port_ids = []
    created_port_ids = []
    ports_in_requested_order = []
    nets_in_requested_order = []
    # 遍历网络请求列表
    for request in ordered_networks:
        # Network lookup for available network_id
        network = None
        for net in nets:
            # 在可用网络列表中找到和申请的网络
            if net['id'] == request.network_id:
                network = net
                break
        # if network_id did not pass validate_networks() and not available
        # here then skip it safely not continuing with a None Network
        # 这个for else语法查了才知道,for中没有break就会走这里
        # 相当于if network is None
        else:
            continue
        # 在网络申请中的网络列表
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
        # 找对应的l2设置port,反正又是去找neutron server建port
        try:
            self._populate_neutron_extension_values(
                context, instance, request.pci_request_id, port_req_body,
                network=network, neutron=neutron,
                bind_host_id=bind_host_id)
            # 申请网络的时候已经有port信息
            if request.port_id:
                port = ports[request.port_id]
                port_client.update_port(port['id'], port_req_body)
                # 已经存在的port列表增加,这样取消申请的时候不会删除
                # 需要传给get_instance_nw_info,这样后面neutron出错也不删除这些端口
                preexisting_port_ids.append(port['id'])
                # 这个列表存放当前请求需要的port,传给后面的get_instance_nw_info
                ports_in_requested_order.append(port['id'])
            # 申请网络的时候没有port信息,创建port
            else:
                created_port = self._create_port(
                        port_client, instance, request.network_id,
                        port_req_body, request.address,
                        security_group_ids, available_macs, dhcp_opts)
                # 这次请求中创建的port列表增加,唯一作用是在出现Exception的可以删除port
                created_port_ids.append(created_port)
                # 这个列表存放当前请求需要的port,传给后面的get_instance_nw_info
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
    # 返回network_info
    return network_model.NetworkInfo([vif for vif in nw_info
                                      if vif['id'] in created_port_ids +
                                      preexisting_port_ids])
```

network_info是一个扩展的列表

```python
class NetworkInfo(list):
    """Stores and manipulates network information for a Nova instance."""

    # NetworkInfo is a list of VIFs

    def fixed_ips(self):
        """Returns all fixed_ips without floating_ips attached."""
        return [ip for vif in self for ip in vif.fixed_ips()]

    def floating_ips(self):
        """Returns all floating_ips."""
        return [ip for vif in self for ip in vif.floating_ips()]

    @classmethod
    def hydrate(cls, network_info):
        if isinstance(network_info, six.string_types):
            network_info = jsonutils.loads(network_info)
        return cls([VIF.hydrate(vif) for vif in network_info])

    def json(self):
        return jsonutils.dumps(self)
```

get_instance_nw_info过程,这里会详细说明代码

```python
    # 这个函数在基类nova.network.base_api.NetworkAPI
    def get_instance_nw_info(self, context, instance, **kwargs):
        """Returns all network info related to an instance."""
        with lockutils.lock('refresh_cache-%s' % instance.uuid):
            # 先加锁,锁住实例,防止中途变动
            result = self._get_instance_nw_info(context, instance, **kwargs)
            # NOTE(comstud): Don't update API cell with new info_cache every
            # time we pull network info for an instance.  The periodic healing
            # of info_cache causes too many cells messages.  Healing the API
            # will happen separately.
            # 这里叫我们不要每次pull网络都去通知cell api
            # 我们不用cell不用管
            update_cells = kwargs.get('update_cells', False)
            # 更新实例缓存信息(包含网网络信息),这里貌似是新生成了一个实例的缓存
            update_instance_cache_with_nw_info(self, context, instance,
                                               nw_info=result,
                                               update_cells=update_cells)
        return result

    def _get_instance_nw_info(self, context, instance, networks=None,
                              port_ids=None, admin_client=None,
                              preexisting_port_ids=None, **kwargs):
        # NOTE(danms): This is an inner method intended to be called
        # by other code that updates instance nwinfo. It *must* be
        # called with the refresh_cache-%(instance_uuid) lock held!
        # 这里是说调用这个函数前需要调用锁锁住refresh_cache-%(instance_uuid)
        LOG.debug('_get_instance_nw_info()', instance=instance)
        # Ensure that we have an up to date copy of the instance info cache.
        # Otherwise multiple requests could collide and cause cache
        # corruption.
        # 确保我们实例信息缓存是最新的
        # 否则多个请求可能会冲突并导致缓存内容错误
        # 这里用了corruption这个词, 应该是想表达旧数据覆盖新数据的意思
        compute_utils.refresh_info_cache_for_instance(context, instance)
        nw_info = self._build_network_info_model(context, instance, networks,
                                                 port_ids, admin_client,
                                                 preexisting_port_ids)
        return network_model.NetworkInfo.hydrate(nw_info)

    def _build_network_info_model(self, context, instance, networks=None,
                                  port_ids=None, admin_client=None,
                                  preexisting_port_ids=None):
        """Return list of ordered VIFs attached to instance.

        :param context: Request context.
        :param instance: Instance we are returning network info for.
               # 要分配网络的实例
        :param networks: List of networks being attached to an instance.
                         If value is None this value will be populated
                         from the existing cached value.
               # 实例链接的网络列表,也就是前面的nets_in_requested_order
        :param port_ids: List of port_ids that are being attached to an
                         instance in order of attachment. If value is None
                         this value will be populated from the existing
                         cached value.
               # 实例网络用的端口,就是前面的ports_in_requested_order
        :param admin_client: A neutron client for the admin context.
        :param preexisting_port_ids: List of port_ids that nova didn't
                        allocate and there shouldn't be deleted when
                        an instance is de-allocated. Supplied list will
                        be added to the cached list of preexisting port
                        IDs for this instance.
              # 预先存在的端口,实例取消的分配的时候不删除的端口,也就是前面preexisting_port_ids
        """

        search_opts = {'tenant_id': instance.project_id,
                       'device_id': instance.uuid, }
        if admin_client is None:
            client = get_client(context, admin=True)
        else:
            client = admin_client

        data = client.list_ports(**search_opts)
        # 获取实例已经存在的port
        current_neutron_ports = data.get('ports', [])
        # 我们创建实例,networks和port_ids肯定不是none
        nw_info_refresh = networks is None and port_ids is None
        # 这里比较付复杂
        networks, port_ids = self._gather_port_ids_and_networks(
                context, instance, networks, port_ids)
        # 创建一个nw_info列表实例
        nw_info = network_model.NetworkInfo()
        # preexisting_port_ids重新设置一下防止有遗漏?
        if preexisting_port_ids is None:
            preexisting_port_ids = []
        preexisting_port_ids = set(
            preexisting_port_ids + self._get_preexisting_port_ids(instance))

        # 列表转字典,port id为key, port的其他信息为value
        current_neutron_port_map = {}
        for current_neutron_port in current_neutron_ports:
            current_neutron_port_map[current_neutron_port['id']] = (
                current_neutron_port)

        for port_id in port_ids:
            current_neutron_port = current_neutron_port_map.get(port_id)
            if current_neutron_port:
                vif_active = False
                if (current_neutron_port['admin_state_up'] is False
                    or current_neutron_port['status'] == 'ACTIVE'):
                    vif_active = True
                # 通过网络端口获取ip
                network_IPs = self._nw_info_get_ips(client,
                                                    current_neutron_port)
                # 通过端口获取子网
                subnets = self._nw_info_get_subnets(context,
                                                    current_neutron_port,
                                                    network_IPs)
                # 生成网卡名
                devname = "tap" + current_neutron_port['id']
                devname = devname[:network_model.NIC_NAME_LEN]

                network, ovs_interfaceid = (
                    self._nw_info_build_network(current_neutron_port,
                                                networks, subnets))
                preserve_on_delete = (current_neutron_port['id'] in
                                      preexisting_port_ids)

                nw_info.append(network_model.VIF(
                    id=current_neutron_port['id'],
                    address=current_neutron_port['mac_address'],
                    network=network,
                    vnic_type=current_neutron_port.get('binding:vnic_type',
                        network_model.VNIC_TYPE_NORMAL),
                    type=current_neutron_port.get('binding:vif_type'),
                    profile=current_neutron_port.get('binding:profile'),
                    details=current_neutron_port.get('binding:vif_details'),
                    ovs_interfaceid=ovs_interfaceid,
                    devname=devname,
                    active=vif_active,
                    preserve_on_delete=preserve_on_delete))

            elif nw_info_refresh:
                LOG.info(_LI('Port %s from network info_cache is no '
                             'longer associated with instance in Neutron. '
                             'Removing from network info_cache.'), port_id,
                         instance=instance)

        return nw_info


```

有兴趣也可以看看_process_requested_networks的请求过程,这里就不说明了

---

回到ComputeManager中


当_build_and_run_instance调用_build_resources获取到正确的network_info后

```python

    self.driver.spawn(context, instance, image_meta,
                      injected_files, admin_password,
                      network_info=network_info,
                      block_device_info=block_device_info)

```

上面的driver是nova.virt.libvirt.driver


可以看到这部分代码已经和nova启动虚拟机比较像了,[详细启动过程参考](http://www.lolizeppelin.com/2017/01/06/nova-intance-start/)最后启动虚拟机的部分代码

```python
def spawn(self, context, instance, image_meta, injected_files,
          admin_password, network_info=None, block_device_info=None):
    disk_info = blockinfo.get_disk_info(CONF.libvirt.virt_type,
                                        instance,
                                        image_meta,
                                        block_device_info)
    self._create_image(context, instance,
                       disk_info['mapping'],
                       network_info=network_info,
                       block_device_info=block_device_info,
                       files=injected_files,
                       admin_pass=admin_password)
    xml = self._get_guest_xml(context, instance, network_info,
                              disk_info, image_meta,
                              block_device_info=block_device_info,
                              write_to_disk=True)
    self._create_domain_and_network(context, xml, instance, network_info,
                                    disk_info,
                                    block_device_info=block_device_info)
    LOG.debug("Instance is running", instance=instance)

    def _wait_for_boot():
        """Called at an interval until the VM is running."""
        state = self.get_info(instance).state

        if state == power_state.RUNNING:
            LOG.info(_LI("Instance spawned successfully."),
                     instance=instance)
            raise loopingcall.LoopingCallDone()

    timer = loopingcall.FixedIntervalLoopingCall(_wait_for_boot)
    timer.start(interval=0.5).wait()
```

[下一篇](http://www.lolizeppelin.com/2016/12/14/neutron-server-create-port/)neutron创建port的过程
