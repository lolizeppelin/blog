---
layout: post
title:  "OpenStack Mitaka从零开始 nova-compute启动虚拟机实例过程"
date:   2017-01-06 15:05:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}


ComputeManager中启动实例的过程

```python
def start_instance(self, context, instance):
    """Starting an instance on this host."""
    # 调用notify 通知状态power_on.start
    self._notify_about_instance_usage(context, instance, "power_on.start")
    # 启动intance
    self._power_on(context, instance)
    # 获取启动结果
    instance.power_state = self._get_power_state(context, instance)
    instance.vm_state = vm_states.ACTIVE
    instance.task_state = None

    # Delete an image(VM snapshot) for a shelved instance
    snapshot_id = instance.system_metadata.get('shelved_image_id')
    if snapshot_id:
        self._delete_snapshot_of_shelved_instance(context, instance,
                                                  snapshot_id)

    # Delete system_metadata for a shelved instance
    compute_utils.remove_shelved_keys_from_system_metadata(instance)

    instance.save(expected_task_state=task_states.POWERING_ON)
    # 通知状态 power_on.end
    self._notify_about_instance_usage(context, instance, "power_on.end")

    ...
    ...

def _power_on(self, context, instance):
    # 通过network_api获取网络信息,network_api一般是neutronv2.api.API的实例
    network_info = self.network_api.get_instance_nw_info(context, instance)
    # 获取block_device信息,这里只或去系统盘(本地盘)信息,不管远程硬盘
    block_device_info = self._get_instance_block_device_info(context,
                                                             instance)
    # 启动实例...我们kvm用的就是libiver的接口
    self.driver.power_on(context, instance,
                         network_info,
                         block_device_info)
```


get_instance_nw_info实际调用为nova.network.neutronv2.api.API中的_get_instance_nw_info

```python
    def _get_instance_nw_info(self, context, instance, networks=None,
                              port_ids=None, admin_client=None,
                              preexisting_port_ids=None, **kwargs):
        # NOTE(danms): This is an inner method intended to be called
        # by other code that updates instance nwinfo. It *must* be
        # called with the refresh_cache-%(instance_uuid) lock held!
        LOG.debug('_get_instance_nw_info()', instance=instance)
        # Ensure that we have an up to date copy of the instance info cache.
        # Otherwise multiple requests could collide and cause cache
        # corruption.
        compute_utils.refresh_info_cache_for_instance(context, instance)
        # 这里访问neutron返回实例使用的网络的相关信息,子网、网桥、port的id之类
        # 后面nova中生成的libvirt的xml中用的网卡名就是tap加上这里获取的port id
        # 最终网卡名按最大网卡名长度切片, 所用网桥也是这里返回的
        nw_info = self._build_network_info_model(context, instance, networks,
                                                 port_ids, admin_client,
                                                 preexisting_port_ids)
        return network_model.NetworkInfo.hydrate(nw_info)

```




实际执行power on的类为nova.virt.libvirt.driver.LibvirtDriver


```python
def power_on(self, context, instance, network_info,
             block_device_info=None):
    """Power on the specified instance."""
    # We use _hard_reboot here to ensure that all backing files,
    # network, and block device connections, etc. are established
    # and available before we attempt to start the instance.
    self._hard_reboot(context, instance, network_info, block_device_info)

def _hard_reboot(self, context, instance, network_info,
                 block_device_info=None):
    """Reboot a virtual machine, given an instance reference.

    Performs a Libvirt reset (if supported) on the domain.

    If Libvirt reset is unavailable this method actually destroys and
    re-creates the domain to ensure the reboot happens, as the guest
    OS cannot ignore this action.
    """

    self._destroy(instance)

    # Convert the system metadata to image metadata
    instance_dir = libvirt_utils.get_instance_path(instance)
    fileutils.ensure_tree(instance_dir)

    disk_info = blockinfo.get_disk_info(CONF.libvirt.virt_type,
                                        instance,
                                        instance.image_meta,
                                        block_device_info)
    # NOTE(vish): This could generate the wrong device_format if we are
    #             using the raw backend and the images don't exist yet.
    #             The create_images_and_backing below doesn't properly
    #             regenerate raw backend images, however, so when it
    #             does we need to (re)generate the xml after the images
    #             are in place.

    # 生成libvirt的xml, write_to_disk参数控制讲xml内容写入到磁盘(也就是这里重新生成了libvirt用的xml)
    xml = self._get_guest_xml(context, instance, network_info, disk_info,
                              instance.image_meta,
                              block_device_info=block_device_info,
                              write_to_disk=True)

    if context.auth_token is not None:
        # 有token 重新获取一次备用disk的info
        # NOTE (rmk): Re-populate any missing backing files.
        backing_disk_info = self._get_instance_disk_info(instance.name,
                                                         xml,
                                                         block_device_info)
        # 创建备份盘
        self._create_images_and_backing(context, instance, instance_dir,
                                        backing_disk_info)

    # Initialize all the necessary networking, block devices and
    # start the instance.
    # 看英文说明,初始化网络、block devices,然后启动虚拟机
    self._create_domain_and_network(context, xml, instance, network_info,
                                    disk_info,
                                    block_device_info=block_device_info,
                                    reboot=True,
                                    vifs_already_plugged=True)
    # pci设备？
    self._prepare_pci_devices_for_use(
        pci_manager.get_instance_pci_devs(instance, 'all'))

    def _wait_for_reboot():
        """Called at an interval until the VM is running again."""
        state = self.get_info(instance).state

        if state == power_state.RUNNING:
            LOG.info(_LI("Instance rebooted successfully."),
                     instance=instance)
            raise loopingcall.LoopingCallDone()
    # 启一个计时器,调用_wait_for_reboot直到 state == power_state.RUNNING
    timer = loopingcall.FixedIntervalLoopingCall(_wait_for_reboot)
    timer.start(interval=0.5).wait()


    def _create_domain_and_network(self, context, xml, instance, network_info,
                                   disk_info, block_device_info=None,
                                   power_on=True, reboot=False,
                                   vifs_already_plugged=False):

        """Do required network setup and create domain."""
        block_device_mapping = driver.block_device_info_get_mapping(
            block_device_info)

        for vol in block_device_mapping:
            connection_info = vol['connection_info']

            if (not reboot and 'data' in connection_info and
                    'volume_id' in connection_info['data']):
                volume_id = connection_info['data']['volume_id']
                encryption = encryptors.get_encryption_metadata(
                    context, self._volume_api, volume_id, connection_info)

                if encryption:
                    encryptor = self._get_volume_encryptor(connection_info,
                                                           encryption)
                    encryptor.attach_volume(context, **encryption)

        timeout = CONF.vif_plugging_timeout
        # _get_neutron_events是所有未激活的port id组成的事件列表
        # 事件名称是network-vif-plugged
        # 这些信息是之前找neutron获取的
        if (self._conn_supports_start_paused and
            utils.is_neutron() and not
            vifs_already_plugged and power_on and timeout):
            events = self._get_neutron_events(network_info)
        else:
            events = []

        pause = bool(events)
        guest = None
        try:
            # 这里做了个定时通知事件
            # 在设置防火墙、并启动虚拟机以后
            # 会检测未激活的port id事件直到超时
            # 这个event部分的代码只知道大概意思,反正我是没看明白
            # 装饰器看着烦
            with self.virtapi.wait_for_instance_event(
                    instance, events, deadline=timeout,
                    error_callback=self._neutron_failed_callback):
                # 插入虚拟网卡, 当我们使用openvswitch的时候
                # 这行代码没有任何功能
                # 因为是直接配置在xml里,qeum启动的时候自动处理
                self.plug_vifs(instance, network_info)
                # 设置基础防火墙规则
                self.firewall_driver.setup_basic_filtering(instance,
                                                           network_info)
                # 执行生成iptable贵规则
                self.firewall_driver.prepare_instance_filter(instance,
                                                             network_info)
                # 我们不是lxc,会直接到后面_create_domain                                            
                with self._lxc_disk_handler(instance, instance.image_meta,
                                            block_device_info, disk_info):
                    # 这里就是启动虚拟机,也就相当于libvirt里调用start
                    guest = self._create_domain(
                        xml, pause=pause, power_on=power_on)
                # 设置防火墙
                self.firewall_driver.apply_instance_filter(instance,
                                                           network_info)
        # 后面是捕获错误,例如前面的网卡插入报错、event执行检测到超时之类的
        # 出错了就poweroff
        except exception.VirtualInterfaceCreateException:
            # Neutron reported failure and we didn't swallow it, so
            # bail here
            with excutils.save_and_reraise_exception():
                if guest:
                    guest.poweroff()
                self.cleanup(context, instance, network_info=network_info,
                             block_device_info=block_device_info)
        except eventlet.timeout.Timeout:
            # We never heard from Neutron
            LOG.warn(_LW('Timeout waiting for vif plugging callback for '
                         'instance %(uuid)s'), {'uuid': instance.uuid},
                     instance=instance)
            if CONF.vif_plugging_is_fatal:
                if guest:
                    guest.poweroff()
                self.cleanup(context, instance, network_info=network_info,
                             block_device_info=block_device_info)
                raise exception.VirtualInterfaceCreateException()

        # Resume only if domain has been paused
        if pause:
            guest.resume()
        return guest

```


总结下

    1、使用openvswitch的时候,虚拟机的网卡最终是novacompute创建xml后,qemu启动虚拟机并绑定到openvswitch的网桥上
    2、如果不用openvswitch的时候网卡也是nove通过各种pulg函数生成网卡的
    3、虚拟机启动的时候一个通过一个事件监控虚拟机的网卡事件,超时就power off
