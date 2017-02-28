---
layout: post
title:  "OpenStack Mitaka从零开始 nova通过neutron分配网络的过程(3)"
date:   2016-12-14 15:05:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}


我们来看看给port分配ip的过程

neutron.db.ipam_pluggable_backend.IpamNonPluggableBackend

```python

class IpamNonPluggableBackend(ipam_backend_mixin.IpamBackendMixin):
    ...
    def allocate_ips_for_port_and_store(self, context, port, port_id):
         network_id = port['port']['network_id']
         # 这里返回子网列表,多个子网只有一条
         ips = self._allocate_ips_for_port(context, port)
         if ips:
             for ip in ips:
                 ip_address = ip['ip_address']
                 subnet_id = ip['subnet_id']
                 self._store_ip_allocation(context, ip_address, network_id,
                                           subnet_id, port_id)
         return ips


     def _allocate_ips_for_port(self, context, port):
         p = port['port']
         ips = []
         v6_stateless = []
         net_id_filter = {'network_id': [p['network_id']]}
         # 这里通过networkid获取这network id下的所有子网
         subnets = self._get_subnets(context, filters=net_id_filter)
         # 判断这个port是路由用的port
         is_router_port = (
             p['device_owner'] in constants.ROUTER_INTERFACE_OWNERS_SNAT)
         # 这里判断是否已经指定了ip信息
         fixed_configured = p['fixed_ips'] is not attributes.ATTR_NOT_SPECIFIED
         # 从上篇文章的port_data字典我们可以看到,port_data是不带fixed_ips字段的
         # 所有不会走这里
         if fixed_configured:
             ....
         else:
             # Split into v4, v6 stateless and v6 stateful subnets
             v4 = []
             v6_stateful = []
             for subnet in subnets:
                 if subnet['ip_version'] == 4:
                     v4.append(subnet)
                 # 后面部分代码都是特殊处理v6 ip的
                 elif ipv6_utils.is_auto_address_subnet(subnet):
                     if not is_router_port:
                         v6_stateless.append(subnet)
                 else:
                     v6_stateful.append(subnet)
             #  version_subnets就是所有的子网信息
             version_subnets = [v4, v6_stateful]
             # 我们ipv4就只有一个v4列表,不管v6
             for subnets in version_subnets:
                  # v4 子网列表
                 if subnets:
                     # 这里把所有v4子网传入,
                     # 然后返回第一个找到的能分配到ip的子网
                     result = IpamNonPluggableBackend._generate_ip(context,
                                                                   subnets)
                     ips.append({'ip_address': result['ip_address'],
                                 'subnet_id': result['subnet_id']})
         # ipv6 静态地址
         for subnet in v6_stateless:
             # IP addresses for IPv6 SLAAC and DHCPv6-stateless subnets
             # are implicitly included.
             ip_address = self._calculate_ipv6_eui64_addr(context, subnet,
                                                          p['mac_address'])
             ips.append({'ip_address': ip_address.format(),
                         'subnet_id': subnet['id']})
         # ips用列表的原因就是,可以同时分配 v4 和 v6 子网
         # 我们只有ipv4的话, ips列表将只有一条记录
         # 完整结构为
         #  [{ipv4 子网}， {自动分配ivp6 子网}, {静态ipv6子网}]
         return ips

     @staticmethod
     def _generate_ip(context, subnets):
         try:
             # 可以看到这里只要子网可以分配到ip,就会返回
             # 所以有多个子网也只会返回一个
             return IpamNonPluggableBackend._try_generate_ip(context, subnets)
         except n_exc.IpAddressGenerationFailure:
             IpamNonPluggableBackend._rebuild_availability_ranges(context,
                                                                  subnets)
         # 这里递归,不停的找有可用ip的子网
         return IpamNonPluggableBackend._try_generate_ip(context, subnets)

```
