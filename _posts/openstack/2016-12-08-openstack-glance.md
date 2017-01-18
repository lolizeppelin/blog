---
layout: post
title:  "OpenStack Mitaka从零开始 glance服务"
date:   2016-12-08 15:05:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}


## nova-api需要配置glance服务,所以我们先配置glance

#### glance是干什么的
1. 公有云里中,安装镜像(centos windows之类的各种镜像)由glance提供
2. 快照的生成的image文件是通过glance里存放的
3. glance可以将上述的镜像存放在本地硬盘,但是有一定规模后glance本地硬盘肯定不够用,所以glance一般是用swift(或者Ceph文件系统)来存放镜像,我们做openstack入门可以先不碰swift/Ceph,所以glance的使用本地硬盘存放镜像即可,配置中swift相关的也可以忽略

#### glance带了4个服务
    # 主要服务,对外api, 分为v1 v2
    /usr/lib/systemd/system/openstack-glance-api.service
    # 主要服务,对内api 分v1 v2 (外部请求访问api,api将请求转化访问registry,所有api和registry都要启动)
    /usr/lib/systemd/system/openstack-glance-registry.service
    # 次要服务,K版中说明是新分出一个项目Glare，专门提供特殊的源服务。
    # 目前尚在开发中，希望通过该项目可以完善各种App Store的安装问题。
    # 看样子在M版里这个服务是完工了的,目前对我们没什么用忽略
    /usr/lib/systemd/system/openstack-glance-glare.service
    # 次要服务,一个定时器,启用镜像延迟删除后,定时清理过期镜像
    /usr/lib/systemd/system/openstack-glance-scrubber.service


#### 简要glance-api.service配置文件

```config_file
[DEFAULT]
enable_v1_api = false
enable_v2_api = true
enable_v1_registry = false
enable_v2_registry = false
location_strategy = location_order
# registry位置
registry_host = 127.0.0.1
registry_port = 9191
# 是否把认证token发送给registry,不发送的话registry还要去keysone再认证一次
# 顺便,这里可以设置认证的相关信息,也可以通过registry的配置文件获取
send_identity_headers = true
# rpc对应信息
rpc_backend = rabbit
control_exchange = glance
rpc_response_timeout = 60
executor_thread_pool_size = 64

[database]
backend = sqlalchemy
connection = <None>
# 这个参数控制是否用绿色线程来管理mysql连接池
# use_tpool = false

[glance_store]
stores = file
default_store = file
# 存储方式使用file的的时候,本地存放的镜像的位置
filesystem_store_datadir =  /home/glance/image

# 如果下面的flavor不走和keystone认证的pipline
# 这里可以不配置任何内容
[keystone_authtoken]
auth_uri = your keystone public url
auth_version = 3.0
delay_auth_decision = false
http_connect_timeout = 5
http_request_max_retries = 3
region_name = 1008
memcached_servers = 192.168.2.202:11211

[oslo_messaging_notifications]
driver = messagingv2

[oslo_messaging_rabbit]
rabbit_host = 192.168.2.202
rabbit_port = 5673
rabbit_userid = openstack
rabbit_password = openstack
rabbit_virtual_host = openstack

[paste_deploy]
# flavor是个比较关键的配置
# 这个配置定义了你走glance-api-paste.ini中对应的pipeline
# 这个flavor对应的pipeline是使用了keystone做二次认证的流程
# 不使用镜像缓存
flavor = keystone
config_file = glance-api-paste.ini的

[task]
# 当task type是base import的时候
# 这个配置会在filesystem_store_datadir上设置override
work_dir = /home/glance/work
```
---

#### 附加说明

    glance的默认端口写在代码里,配置文件里反而没有默认值
    glance的cache不是放token的cache
    glance存放的镜像文件可以是本地的,可以是http的可以是swift的
    nova找glance要image的是后, glane作为一个中间人传输文件
    为了降低传输消耗,glance会缓存热点的image到本地
    如果你glance只是测试用,image只放在本地,当然就不需要用到cache来存放镜像
    因为镜像本来就在本地

    glance和客户端(租户)的token没有关系,memcache可以是不同的也可以相同
    glance要缓存token只需要配置memcache服务器就可以了
    token不需要设置前缀,代码里有让key唯一的方法

glacne的入口,这里可以看出我们的访问url是/v2, app是glance.api.v2.router:API.factory

```config_file
[pipeline:glance-api-keystone]
pipeline = cors healthcheck versionnegotiation osprofiler authtoken context  rootapp

[composite:rootapp]
paste.composite_factory = glance.api:root_app_factory
/: apiversions
/v1: apiv1app
/v2: apiv2app

[app:apiv2app]
paste.app_factory = glance.api.v2.router:API.factory

```

router定位到

```python
class API(wsgi.Router):

    """WSGI router for Glance v2 API requests."""

    def __init__(self, mapper):
        custom_image_properties = images.load_custom_properties()
        reject_method_resource = wsgi.Resource(wsgi.RejectMethodController())
```

controller最后定位到

```python
class Controller(object):
    def __init__(self, custom_image_properties=None):
        self.image_schema = images.get_schema(custom_image_properties)
        self.image_collection_schema = images.get_collection_schema(
            custom_image_properties)
        self.member_schema = image_members.get_schema()
        self.member_collection_schema = image_members.get_collection_schema()
        self.task_schema = tasks.get_task_schema()
        self.task_collection_schema = tasks.get_collection_schema()

        # Metadef schemas
        self.metadef_namespace_schema = metadef_namespaces.get_schema()
        self.metadef_namespace_collection_schema = (
            metadef_namespaces.get_collection_schema())

        self.metadef_resource_type_schema = metadef_resource_types.get_schema()
        self.metadef_resource_type_collection_schema = (
            metadef_resource_types.get_collection_schema())

        self.metadef_property_schema = metadef_properties.get_schema()
        self.metadef_property_collection_schema = (
            metadef_properties.get_collection_schema())

        self.metadef_object_schema = metadef_objects.get_schema()
        self.metadef_object_collection_schema = (
            metadef_objects.get_collection_schema())

        self.metadef_tag_schema = metadef_tags.get_schema()
        self.metadef_tag_collection_schema = (
            metadef_tags.get_collection_schema())
```                                       

custom_image_properties是

    openstack的镜像管理模块glance，采用了一个名为image_properties的表来记录各镜像的
    属性信息，例如该某个镜像使用的系统架构（architecture）、操作系统的版本等
[image_properties参考](http://blog.csdn.net/weiyuanke/article/details/23788875)

openstack执行命令添加endpoint

    (openstack) endpoint create --region 1008 glance internal http://openstack.szdiyibo.com:9292/v2
    (openstack) endpoint create --region 1008 glance admin http://openstack.szdiyibo.com:9292/v2
    (openstack) endpoint create --region 1008 glance public http://openstack.szdiyibo.com:9292/v2
