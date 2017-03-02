---
layout: post
title:  "OpenStack Mitaka从零开始 neutron server的初始化"
date:   2017-02-24 15:05:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}


####  neutron server的初始化从APIRouter开始


```python

# neutron.api.v2.router.py
# resource和collection 名的映射字典
RESOURCES = {'network': 'networks',
             'subnet': 'subnets',
             'subnetpool': 'subnetpools',
             'port': 'ports'}

class APIRouter(base_wsgi.Router):

    def __init__(self, **local_config):
        mapper = routes_mapper.Mapper()
        # neutron.manager.py
        # get_plugin()返回的是NeutronManager实例(单例)的属性plugin
        # 也就是 CORE_PLUGINS
        # neutron.plugins.ml2.plugin.py中的Ml2Plugin
        plugin = manager.NeutronManager.get_plugin()
        ext_mgr = extensions.PluginAwareExtensionManager.get_instance()
        ext_mgr.extend_resources("2.0", attributes.RESOURCE_ATTRIBUTE_MAP)

        col_kwargs = dict(collection_actions=COLLECTION_ACTIONS,
                          member_actions=MEMBER_ACTIONS)

        def _map_resource(collection, resource, params, parent=None):
            allow_bulk = cfg.CONF.allow_bulk
            allow_pagination = cfg.CONF.allow_pagination
            allow_sorting = cfg.CONF.allow_sorting
            # 看后面的create_resource代码部分说明
            controller = base.create_resource(
                collection, resource, plugin, params, allow_bulk=allow_bulk,
                parent=parent, allow_pagination=allow_pagination,
                allow_sorting=allow_sorting)
            path_prefix = None
            # 没有设置parent,不传入path_prefix会有默认path_prefix
            if parent:
                # 这里代码不能贴...贴了gitblog发布不了
                path_prefix = xxxxx

            mapper_kwargs = dict(controller=controller,
                                 requirements=REQUIREMENTS,
                                 path_prefix=path_prefix,
                                 **col_kwargs)
            # 映射资源到mapper上
            return mapper.collection(collection, resource,
                                     **mapper_kwargs)

        mapper.connect('index', '/', controller=Index(RESOURCES))

        # 调用_map_resource注册resource
        for resource in RESOURCES:
            # 一个个controller注册进去
            _map_resource(RESOURCES[resource], resource,
                          attributes.RESOURCE_ATTRIBUTE_MAP.get(
                              RESOURCES[resource], dict()))
            resource_registry.register_resource_by_name(resource)

        # 注册额外的resource
        for resoue in SUB_RESOURCES:
            _map_resource(SUB_RESOURCES[resource]['collection_name'], resource,
                          attributes.RESOURCE_ATTRIBUTE_MAP.get(
                              SUB_RESOURCES[resource]['collection_name'],
                              dict()),
                          SUB_RESOURCES[resource]['parent'])
        policy.reset()
        super(APIRouter, self).__init__(mapper)
```

这里是controller生成并转换为wsig的app的部分

```python
def create_resource(collection, resource, plugin, params, allow_bulk=False,
                    member_actions=None, parent=None, allow_pagination=False,
                    allow_sorting=False):
    # 这里就是具体的controller,具体的执行代码就在controller中
    controller = Controller(plugin, collection, resource, params, allow_bulk,
                            member_actions=member_actions, parent=parent,
                            allow_pagination=allow_pagination,
                            allow_sorting=allow_sorting)
    # 把controller转为wsig的app
    return wsgi_resource.Resource(controller, FAULT_MAP)
```
