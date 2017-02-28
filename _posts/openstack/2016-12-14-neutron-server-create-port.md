---
layout: post
title:  "OpenStack Mitaka从零开始 nova通过neutron分配网络的过程(2)"
date:   2016-12-14 15:05:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}



neutron server分配创建port

(port的)Controller--Controller.create-->Controller.\_create-->Ml2Plugin.create_port-->Ml2Plugin.\_create_port_db-->Ml2Plugin.create_port_db
-->Ml2Plugin.\_create_port-->Ml2Plugin.\_create_port_with_mac（这里没分配IP但是join了有分配好ip的表,mysql驱动先jion后分配IP）

```python

# neutron.api.v2.base.py
# collection

class Controller(object):
    LIST = 'list'
    SHOW = 'show'
    CREATE = 'create'
    UPDATE = 'update'
    DELETE = 'delete'

    def __init__(self, plugin, collection, resource, attr_info,
                 allow_bulk=False, member_actions=None, parent=None,
                 allow_pagination=False, allow_sorting=False):
        # 所有的Controller  plugin都是
        # neutron.plugins.ml2.plugin.Ml2Plugin
        # port的Controller中
        # collection: "ports"
        # resource: "port"
        # parent: None  默认resource的parent都是None,一个resource一个Controller
        # attr_info的值,
        # PORTS: {
        #     'id': {'allow_post': False, 'allow_put': False,
        #            'validate': {'type:uuid': None},
        #            'is_visible': True,
        #            'primary_key': True},
        #     'name': {'allow_post': True, 'allow_put': True, 'default': '',
        #              'validate': {'type:string': NAME_MAX_LEN},
        #              'is_visible': True},
        #     'network_id': {'allow_post': True, 'allow_put': False,
        #                    'required_by_policy': True,
        #                    'validate': {'type:uuid': None},
        #                    'is_visible': True},
        #     'admin_state_up': {'allow_post': True, 'allow_put': True,
        #                        'default': True,
        #                        'convert_to': convert_to_boolean,
        #                        'is_visible': True},
        #     'mac_address': {'allow_post': True, 'allow_put': True,
        #                     'default': ATTR_NOT_SPECIFIED,
        #                     'validate': {'type:mac_address': None},
        #                     'enforce_policy': True,
        #                     'is_visible': True},
        #     'fixed_ips': {'allow_post': True, 'allow_put': True,
        #                   'default': ATTR_NOT_SPECIFIED,
        #                   'convert_list_to': convert_kvp_list_to_dict,
        #                   'validate': {'type:fixed_ips': None},
        #                   'enforce_policy': True,
        #                   'is_visible': True},
        #     'device_id': {'allow_post': True, 'allow_put': True,
        #                   'validate': {'type:string': DEVICE_ID_MAX_LEN},
        #                   'default': '',
        #                   'is_visible': True},
        #     'device_owner': {'allow_post': True, 'allow_put': True,
        #                      'validate': {'type:string': DEVICE_OWNER_MAX_LEN},
        #                      'default': '', 'enforce_policy': True,
        #                      'is_visible': True},
        #     'tenant_id': {'allow_post': True, 'allow_put': False,
        #                   'validate': {'type:string': TENANT_ID_MAX_LEN},
        #                   'required_by_policy': True,
        #                   'is_visible': True},
        #     'status': {'allow_post': False, 'allow_put': False,
        #                'is_visible': True},
        # },


        if member_actions is None:
            member_actions = []
        self._plugin = plugin
        self._collection = collection.replace('-', '_') # ports
        self._resource = resource.replace('-', '_') # port
        ...
        # _attr_info中给 policy用的参数
        self._policy_attrs = [name for (name, info) in self._attr_info.items()
                              if info.get('required_by_policy')]
        ...
        if parent:
            self._parent_id_name = '%s_id' % parent['member_name']
            parent_part = '_%s' % parent['member_name']
        else:
            self._parent_id_name = None
            parent_part = ''
        self._plugin_handlers = {
            self.LIST: 'get%s_%s' % (parent_part, self._collection),
            self.SHOW: 'get%s_%s' % (parent_part, self._resource)
        }

        for action in [self.CREATE, self.UPDATE, self.DELETE]:
            self._plugin_handlers[action] = '%s%s_%s' % (action, parent_part,
                                                         self._resource)

        # self._plugin_handlers = {'list': 'get_ports',
        #                          'show': 'get_port',
        #                          'create': 'create_port',
        #                          'update': 'update_port',
        #                          'delete': 'delete_port'}


    def create(self, request, body=None, **kwargs):
        self._notifier.info(request.context,
                            self._resource + '.create.start',
                            body)
        return self._create(request, body, **kwargs)


    def _create(self, request, body, **kwargs):
        """Creates a new instance of the requested entity."""
        parent_id = kwargs.get(self._parent_id_name)
        body = Controller.prepare_request_body(request.context,
                                               copy.deepcopy(body), True,
                                               self._resource, self._attr_info,
                                               allow_bulk=self._allow_bulk)
        action = self._plugin_handlers[self.CREATE]
        # action = create_port
        # Check authz
        if self._collection in body:
            # Have to account for bulk create
            items = body[self._collection]
        else:
            items = [body]
        # Ensure policy engine is initialized
        policy.init()
        # Store requested resource amounts grouping them by tenant
        # This won't work with multiple resources. However because of the
        # current structure of this controller there will hardly be more than
        # one resource for which reservations are being made
        request_deltas = collections.defaultdict(int)
        for item in items:
            self._validate_network_tenant_ownership(request,
                                                    item[self._resource])
            policy.enforce(request.context,
                           action,
                           item[self._resource],
                           pluralized=self._collection)
            if 'tenant_id' not in item[self._resource]:
                # no tenant_id - no quota check
                continue
            tenant_id = item[self._resource]['tenant_id']
            request_deltas[tenant_id] += 1
        # Quota enforcement
        reservations = []
        try:
            for (tenant, delta) in request_deltas.items():
                reservation = quota.QUOTAS.make_reservation(
                    request.context,
                    tenant,
                    {self._resource: delta},
                    self._plugin)
                reservations.append(reservation)
        except exceptions.QuotaResourceUnknown as e:
                # We don't want to quota this resource
                LOG.debug(e)

        def notify(create_result):
            # Ensure usage trackers for all resources affected by this API
            # operation are marked as dirty
            with request.context.session.begin():
                # Commit the reservation(s)
                for reservation in reservations:
                    quota.QUOTAS.commit_reservation(
                        request.context, reservation.reservation_id)
                resource_registry.set_resources_dirty(request.context)

            notifier_method = self._resource + '.create.end'
            self._notifier.info(request.context,
                                notifier_method,
                                create_result)
            self._send_dhcp_notification(request.context,
                                         create_result,
                                         notifier_method)
            return create_result

        def do_create(body, bulk=False, emulated=False):
            kwargs = {self._parent_id_name: parent_id} if parent_id else {}
            if bulk and not emulated:
                obj_creator = getattr(self._plugin, "%s_bulk" % action)
            else:
                # action = create_port
                # obj_creator = Ml2Plugin.create_port
                obj_creator = getattr(self._plugin, action)
            try:
                if emulated:
                    return self._emulate_bulk_create(obj_creator, request,
                                                     body, parent_id)
                else:
                    if self._collection in body:
                        # This is weird but fixing it requires changes to the
                        # plugin interface
                        kwargs.update({self._collection: body})
                    else:
                        kwargs.update({self._resource: body})

                    # 这里就是下面的 Ml2Plugin.create_port(context, port)
                    # kwargs = {'port': body}
                    # 也就是Ml2Plugin.create_port的第二个参数port = {'port': body}
                    return obj_creator(request.context, **kwargs)
                    # 我们跳去看neutron.plugins.ml2.plugin.Ml2Plugin.create_port
            except Exception:
                # In case of failure the plugin will always raise an
                # exception. Cancel the reservation
                with excutils.save_and_reraise_exception():
                    for reservation in reservations:
                        quota.QUOTAS.cancel_reservation(
                            request.context, reservation.reservation_id)

        if self._collection in body and self._native_bulk:
            # plugin does atomic bulk create operations
            objs = do_create(body, bulk=True)
            # Use first element of list to discriminate attributes which
            # should be removed because of authZ policies
            fields_to_strip = self._exclude_attributes_by_policy(
                request.context, objs[0])
            return notify({self._collection: [self._filter_attributes(
                request.context, obj, fields_to_strip=fields_to_strip)
                for obj in objs]})
        else:
            if self._collection in body:
                # Emulate atomic bulk behavior
                objs = do_create(body, bulk=True, emulated=True)
                return notify({self._collection: objs})
            else:
                obj = do_create(body)
                self._send_nova_notification(action, {},
                                             {self._resource: obj})
                return notify({self._resource: self._view(request.context,
                                                          obj)})

```

创建Port

```python
# 上面的self._pluging 就是 neutron.plugins.ml2.plugin.py
# 中的 Ml2Plugin

class Ml2Plugin(db_base_plugin_v2.NeutronDbPluginV2,
                dvr_mac_db.DVRDbMixin,
                external_net_db.External_net_db_mixin,
                sg_db_rpc.SecurityGroupServerRpcMixin,
                agentschedulers_db.AZDhcpAgentSchedulerDbMixin,
                addr_pair_db.AllowedAddressPairsMixin,
                vlantransparent_db.Vlantransparent_db_mixin,
                extradhcpopt_db.ExtraDhcpOptMixin,
                netmtu_db.Netmtu_db_mixin,
                address_scope_db.AddressScopeDbMixin):

    .....
    # 这里继承自neutron.db.db_base_plugin_v2.NeutronDbPluginV2
    def create_port_db(self, context, port):
        # 这里可以看,create_port_db和_create_port_db不是一个人写的
        # _create_port_db用的是attrs = port[attributes.PORT]
        # 这里用的是p = port['port']
        # 这里的p就是传进来的body
        p = port['port']
        port_id = p.get('id') or uuidutils.generate_uuid()
        network_id = p['network_id']
        # NOTE(jkoelker) Get the tenant_id outside of the session to avoid
        #                unneeded db action if the operation raises
        tenant_id = p['tenant_id']
        if p.get('device_owner'):
            self._enforce_device_owner_not_router_intf_or_device_id(
                context, p.get('device_owner'), p.get('device_id'), tenant_id)

        port_data = dict(tenant_id=tenant_id,
                         name=p['name'],
                         id=port_id,
                         network_id=network_id,
                         admin_state_up=p['admin_state_up'],
                         status=p.get('status', constants.PORT_STATUS_ACTIVE),
                         device_id=p['device_id'],
                         device_owner=p['device_owner'],
                         description=p.get('description'))
        if ('dns-integration' in self.supported_extension_aliases and
            'dns_name' in p):
            request_dns_name = self._get_request_dns_name(p)
            port_data['dns_name'] = request_dns_name

        with context.session.begin(subtransactions=True):
            # Ensure that the network exists.
            # 通过network_id查找network来确认network存在
            self._get_network(context, network_id)

            # Create the port
            # 通过port_data创建端口
            # port_data中只有network_id没有fixed_ips
            if p['mac_address'] is attributes.ATTR_NOT_SPECIFIED:
                # crate_port最终会调用_create_port_with_mac
                db_port = self._create_port(context, network_id, port_data)
                p['mac_address'] = db_port['mac_address']
            else:
                # _create_port_with_mac创建neutron.db.models_v2.Port实例
                # 这个实例目前的fixed_ips是空列表
                db_port = self._create_port_with_mac(
                    context, network_id, port_data, p['mac_address'])
            # 这里ips结构为, ipv4只返回第一个找到的有可用ip的子网
            # [{'subnet_id': u'1cabecae-d7c8-4a25-b2a1-758df2619667',
            #   'ip_address': u'192.168.1.43'}]
            # 也就是说到这一步才分配到了ip
            # 问题在于,先连表(join)再创建的原理  得看sqlalchemy的orm部分代码才懂
            # 总之,ips是allocate_ips_for_port_and_store创建的
            # ipam就是 neutron.db.ipam_pluggable_backend.IpamPluggableBackend
            ips = self.ipam.allocate_ips_for_port_and_store(context, port,
                                                            port_id)
            if ('dns-integration' in self.supported_extension_aliases and
                'dns_name' in p):
                dns_assignment = []
                if ips:
                    dns_assignment = self._get_dns_names_for_port(
                        context, ips, request_dns_name)
                db_port['dns_assignment'] = dns_assignment
        # db_port是一个neutron.db.models_v2.Port实例,
        # 对应到数据库的ports表
        # 实例默认的 fixed_ips属性是 port表 以port_id
        # join ipallocations表获得的列表
        # 我们port所用的ip信息就在ipallocations表中
        return db_port


    def _create_port_db(self, context, port):
        # 这里的attrs就是传进来的body
        attrs = port[attributes.PORT]
        if not attrs.get('status'):
            attrs['status'] = const.PORT_STATUS_DOWN

        session = context.session
        with db_api.exc_to_retry(os_db_exception.DBDuplicateEntry),\
                session.begin(subtransactions=True):
            dhcp_opts = attrs.get(edo_ext.EXTRADHCPOPTS, [])
            # body字典中的内容转换为
            # neutron.db.models_v2.Port实例
            # port_db的fixed_ips属性就是我们创建port后分配到ip列表
            port_db = self.create_port_db(context, port)
            # Port实例转成dict
            result = self._make_port_dict(port_db, process_extensions=False)
            self.extension_manager.process_create_port(context, attrs, result)
            self._portsec_ext_port_create_processing(context, result, port)

            # sgids must be got after portsec checked with security group
            sgids = self._get_security_groups_on_port(context, port)
            self._process_port_create_security_group(context, result, sgids)
            network = self.get_network(context, result['network_id'])
            binding = db.add_port_binding(session, result['id'])
            mech_context = driver_context.PortContext(self, context, result,
                                                      network, binding, None)
            self._process_port_binding(mech_context, attrs)

            result[addr_pair.ADDRESS_PAIRS] = (
                self._process_create_allowed_address_pairs(
                    context, result,
                    attrs.get(addr_pair.ADDRESS_PAIRS)))
            self._process_port_create_extra_dhcp_opts(context, result,
                                                      dhcp_opts)
            self.mechanism_manager.create_port_precommit(mech_context)

        self._apply_dict_extend_functions('ports', result, port_db)
        return result, mech_context

    def create_port(self, context, port):
        # 外面的 obj_creator(request.context, **kwargs)
        # TODO(kevinbenton): remove when bug/1543094 is fixed.
        with lockutils.lock(port['port']['network_id'],
                            lock_file_prefix='neutron-create-port',
                            external=True):
            result, mech_context = self._create_port_db(context, port)
        # notify any plugin that is interested in port create events
        kwargs = {'context': context, 'port': result}
        registry.notify(resources.PORT, events.AFTER_CREATE, self, **kwargs)

        try:
            # commit之前,看上去像是在加锁
            self.mechanism_manager.create_port_postcommit(mech_context)
        except ml2_exc.MechanismDriverError:
            with excutils.save_and_reraise_exception():
                LOG.error(_LE("mechanism_manager.create_port_postcommit "
                              "failed, deleting port '%s'"), result['id'])
                self.delete_port(context, result['id'])

        # REVISIT(rkukura): Is there any point in calling this before
        # a binding has been successfully established?
        self.notify_security_groups_member_updated(context, result)

        try:
            bound_context = self._bind_port_if_needed(mech_context)
        except os_db_exception.DBDeadlock:
            # bind port can deadlock in normal operation so we just cleanup
            # the port and let the API retry
            with excutils.save_and_reraise_exception():
                LOG.debug("_bind_port_if_needed deadlock, deleting port %s",
                          result['id'])
                self.delete_port(context, result['id'])
        except ml2_exc.MechanismDriverError:
            with excutils.save_and_reraise_exception():
                LOG.error(_LE("_bind_port_if_needed "
                              "failed, deleting port '%s'"), result['id'])
                self.delete_port(context, result['id'])

        return bound_context.current

```

接下来的IP分配参考nova通过neutron分配网络的过程(3)
