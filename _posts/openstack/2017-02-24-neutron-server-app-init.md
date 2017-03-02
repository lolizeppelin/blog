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

class APIRouter(base_wsgi.Router):
    def __init__(self, **local_config):
        mapper = routes_mapper.Mapper()
        plugin = manager.NeutronManager.get_plugin()
        ext_mgr = extensions.PluginAwareExtensionManager.get_instance()
        ext_mgr.extend_resources("2.0", attributes.RESOURCE_ATTRIBUTE_MAP)

```
