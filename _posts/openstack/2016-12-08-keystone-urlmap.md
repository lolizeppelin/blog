---
layout: post
title:  "OpenStack Mitaka从零开始 通过keystone理解urlmap映射过程"
date:   2016-12-08 15:05:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}



## 我们以keystone为例一步步看wsgi服务器的启动与paste.deploy的url映射

先截取一部分ini文件先做大致说明

```config_file
[composite:main]
use = egg:Paste#urlmap
/v2.0 = public_api
/v3 = api_v3
/ = public_version_api

[pipeline:api_v3]
# The last item in this pipeline must be service_v3 or an equivalent
# application. It cannot be a filter.
pipeline = cors sizelimit url_normalize request_id admin_token_auth build_auth_context token_auth json_body ec2_extension_v3 s3_extension service_v3

[filter:token_auth]
use = egg:keystone#token_auth
token_auth = keystone.middleware:TokenAuthMiddleware.factory

[app:service_v3]
use = egg:keystone#service_v3
service_v3 = keystone.version.service:v3_app_factory
```

当一个请求过来的时候,先去composite:main找到对应url path
比如来了一个v3请求, 这时候被映射到pipeline:api_v3,
然后这个请求会被pipeline:api_v3列表中的所有filter处理一遍,里面每个filter都要定义,
pipeline的最后的一个是为

    service_v3
在paste.ini里他指向一个app

    [app:service_v3]
    use = egg:keystone#service_v3

问题1：名字相同怎么办——不会存在名字相同,否则会在启动时报错

问题2：这玩意怎么工作的——这就比较复杂了

urlmap映射用的是paste.deploy,
paste是一个包,包名叫Paste,rpm名字为python-paste,可以yum,
deploy是paste的扩展包,包名字叫PasteDeploy,这玩意要自己打包,版本很久没更新了,最新1.5.2,
附上spec文件（openstack用的旧版）

```config_file
%define python_sitearch %(%{__python} -c "from distutils.sysconfig import get_python_lib; print get_python_lib(0)")

Name: python-PasteDeploy
Version: 1.5.2
Release: 0%{?dist}
Summary: Load, configure, and compose WSGI applications and servers
Group: Libraries/Python
License: Python
URL: http://pythonpaste.org/deploy/
Source0: PasteDeploy-%{version}.tar.gz
BuildArch: noarch
BuildRequires: python-devel
BuildRequires: python-setuptools >= 0.6-0.a9.1
Requires: python >= 2.6
Requires: python-paste

%description
This tool provides code to load WSGI applications and servers from
URIs; these URIs can refer to Python Eggs for INI-style configuration
files. Paste Script provides commands to serve applications based on
this configuration file.

%prep
%setup -q -n PasteDeploy-%{version}


%build
CFLAGS="$RPM_OPT_FLAGS" %{__python} setup.py build


%install
%{__python} setup.py install -O1 --skip-build --root %{buildroot}
install -D -m 644 README %{buildroot}%{_docdir}/%{name}-%{version}
install -D -m 644 PKG-INFO %{buildroot}%{_docdir}/%{name}-%{version}
install -D -m 644 MANIFEST.in %{buildroot}%{_docdir}/%{name}-%{version}
#install -D -m 644 docs/news.txt %{buildroot}%{_docdir}/%{name}-%{version}
#install -D -m 644 docs/license.txt.in %{buildroot}%{_docdir}/%{name}-%{version}
#install -D -m 644 doc/index.txt %{buildroot}%{_docdir}/%{name}-%{version}


%files
%doc README PKG-INFO MANIFEST.in
%{python_sitearch}/paste/deploy
%{python_sitearch}/PasteDeploy-%{version}-py*.egg-info
%{python_sitearch}/PasteDeploy-%{version}-py*-nspkg.pth

%changelog
* Sat Aug 20 2016 gcy - 1.5.2
- gcy build it for Redhat EL 6
```

    paste.deploy这玩意比较蛋痛,通过egg来动态载入模块(openstack里自己的模块使用import util)
    所以必须安装setuptool,并要求python的包里带有egg信息
    因为这玩意load的时候,都是通过包内egg-info下的entry_points.txt来映射的
    用起来有点像java的struct, 所以python的rpm包的时候必须带入egg info


### deploy中有3种load函数

    loadapp
    配置中的composite:、pipeline:、app:都由他来处理（还有一个filter-app,keyston未使用）
    因为这三个开头的都是用loadapp,所以他们的后面的name不可以相同（会报错）
    比如有了composite:main,就不能再有app:main
---
    loadfilter
    配置中的filter:都由他处理
---
    loadserver
    配置中的server:都由他处理,openstack的各个服务没有用它来管理自己的wsig,都是自己管理
    keystone可以之际uwsig来启动
---

我们现在来看启动过程,
init里直接调用的keyston-all来启动,
实际就是启动两个server

```python
# 通过绿色线程启动两个进程的过程大致如下,count不配置的话直接用cpu的数量
def create_servers():
    admin_worker_count = _get_workers('admin_workers')
    public_worker_count = _get_workers('public_workers')

    servers = []
    servers.append(create_server(paste_config,
                                 'admin',
                                 CONF.eventlet_server.admin_bind_host,
                                 CONF.eventlet_server.admin_port,
                                 admin_worker_count))
    servers.append(create_server(paste_config,
                                 'main',
                                 CONF.eventlet_server.public_bind_host,
                                 CONF.eventlet_server.public_port,
                                 public_worker_count))
    return servers
```
后面

```python
def create_server(conf, name, host, port, workers):
    app = keystone_service.loadapp('config:%s' % conf, name)
```

实际调用

```python
def loadapp(conf, name):
    # NOTE(blk-u): Save the application being loaded in the controllers module.
    # This is similar to how public_app_factory() and v3_app_factory()
    # register the version with the controllers module.
    controllers.latest_app = deploy.loadapp(conf, name=name)
    return controllers.latest_app
```

两个server对应的name分别为admin和main,都使用loadapp启动.
因为调用的是deploy.loadapp,所以会匹配到[composite:main]和[composite:admin]这两段配置

    eventlet的封装里初始化了全局变量, 不存在因为全局变量问题必须多进程.
    public 和admin的 worker count可以多进程也可以单进程

主进程在调用launch_service之前（最终fork之前）先调用了listen,
fork后通过self.launcher = self._child_process(wrap.service)下的launcher.launch_service(service)启动循环.
socket数据接收直接在各个子进程 通过dup_socket = self.socket.dup() 复制出来的socket accept(看下面的说明来理解多进程接受数据).
socket如何接受数据 处理分包粘包都是在eventlet.wsgi的代码中
对于监听一个socket来说，多个进程同时在accept处阻塞，当有一个连接进入，多个进程同时被唤醒，但之间只有一个进程能成功accept，而不会同时有多个进程能拿到该连接对象，操作系统保证了进程操作这个连接的安全性。

    扩展：上述过程，多个进程同时被唤醒，去抢占accept到的资源，这个现象叫“惊群”，
    而根据网上资料，Linux 内核2.6以下，accept响应时只有一个进程accept成功，其他都失败，
    重新阻塞，也就是说所有监听进程同时被内核调度唤醒，这当然会耗费一定的系统资源。
    而2.6以上，则已经不存在惊群现象了，但是由于开发者开发程序时使用了如epoll等异
    这时子进程保存了父进程的文件描述符的所有副本，可以进程跟父进程一样的操作了。

    我之前一直以为复制的socket的fd和普通文件的fd一样会有共同读写问题
    原来操作系统级别已经避免掉这问题了
    因为eventlet中socket accept的时候没有用异步,所以这里的多进程写起来就很简单了
    现在终于搞清楚了,顺便,每个进程中也使用了协程来支持多个请求

#### 接下来我们继续loapp的源码

```python
def loadapp(uri, name=None, **kw):
    return loadobj(APP, uri, name=name, **kw)

def loadobj(object_type, uri, name=None, relative_to=None,
            global_conf=None):
    context = loadcontext(
        object_type, uri, name=name, relative_to=relative_to,
        global_conf=global_conf)
    return context.create()
```

第一次调用loadcontext传入的url,也就是paste_config,
值通过解析conf文件转为   config:/etc/keystone-paste.ini,
name就是上层传入的name,下面我们以name为main做流程解析,
url.split(‘:’) 后通过冒号前面的config找到实际调用函数.

```python
def _loadconfig(object_type, uri, path, name, relative_to,
                global_conf):
```


    上面函数里面生成ConfigLoader类(就是我们常用解析ini文件的类的封装),并调用类中的get_context方法
    get_context会通过传入的name和APP类的config_prefixes获取到section([composite:main])的内容并放入local_conf变量中
    当section是pipeline或者local_conf中有"use"这个key的时候,有对应递归（看这里大致明白pipeline是怎么一级一级的调用filter了）,
    由于local_conf的key里有"use"(use = egg:Paste#urlmap),调用_context_from_use
    _context_from_use内部local_conf.pop弹出use对应values作为name(这次name为egg:Paste#urlmap)再调用get_context

这次又走到了loadcontext中实际函数为

```python
def _loadegg(object_type, uri, spec, name, relative_to,
             global_conf):
```

    生成EggLoaderload类,
    object_type为APP,name是urlmap，spec为Paste
    调用这个类中get_context返回LoaderContext类

get_context中

```python
    entry_point, protocol, ep_name = self.find_egg_entry_point(object_type, name=name)
```
这里通过setuptool的对应工具找到Paste(spec)的egg配置中的paste.composite_factory即（通过APP(object_type)类的egg_protocols去匹配）

```config_file
[paste.composite_factory]
urlmap = paste.urlmap.urlmap_factory
# 可以看到 urlmap映射到的代码位置为paste.urlmap.urlmap_factory
```

回到返回LoaderContext类

    这部分代码里有个hack, LoaderContext的属性loader本来应该是EggLoaderload,
    但是_context_from_use返回前把LoaderContext的loader覆盖为self,也就是ConfigLoader
    _context_from_use在返回LoaderContext之前还处理了下LoaderContext中的protocol
    后面的处理应该是为了pipeline做的

再回到前面的loadobj,这时候最后的create则是LoaderContext中的
```python
def create(self):
    return self.object_type.invoke(self)
```

当object_type为APP时, APP的invoke为
```python
def invoke(self, context):
    if context.protocol in ('paste.composit_factory',
                            'paste.composite_factory'):
        return fix_call(context.object,
                        context.loader, context.global_conf,
                        **context.local_conf)
    elif context.protocol == 'paste.app_factory':
        return fix_call(context.object, context.global_conf, **context.local_conf)
    else:
        assert 0, "Protocol %r unknown" % context.protocol


def fix_call(callable, *args, **kw):
    """
    Call ``callable(*args, **kw)`` fixing any type errors that come out.
    """
    try:
        val = callable(*args, **kw)
    except TypeError:
        exc_info = fix_type_error(None, callable, args, kw)
        reraise(*exc_info)
    return val
```

上面context.object就是就是之前传入的的entry_point,也就是urlmap_factory

```python
def urlmap_factory(loader, global_conf, **local_conf):
    if 'not_found_app' in local_conf:
        not_found_app = local_conf.pop('not_found_app')
    else:
        not_found_app = global_conf.get('not_found_app')
    if not_found_app:
        not_found_app = loader.get_app(not_found_app, global_conf=global_conf)
    urlmap = URLMap(not_found_app=not_found_app)
    for path, app_name in local_conf.items():
        path = parse_path_expression(path)
        app = loader.get_app(app_name, global_conf=global_conf)
        urlmap[path] = app
    return urlmap
```


到这里local_conf还剩下

    /v2.0 = public_api
    /v3 = api_v3
    / = public_version_api
通过ConfigLoader的get_app(注意前面红字部分),完成url的与调用app的map映射(字典)
到这里,path匹配到对应的处理方法完成
urlmap key对应的value作为wsgi的app, 接收wsgi传入的数据
app的具体初始化过程[参考](http://blog.chinaunix.net/uid-23504396-id-5754581.html)


后面就是通过绿色线程(eventlet.wsgi.server)处理监听端口过来的数据
以key(url path)   values(对应的映射方法)形式分发传入的数据到不同的app进行处理


顺便,在nova-api中,配置文件是这样的
```config_file
[app:metaapp]
paste.app_factory = nova.api.metadata.handler:MetadataRequestHandler.factory
```
匹配会走到
    _context_from_explicit
这样就可以不绕egg找一圈直匹配到对应的类了

至于为什么keystone里用use = egg
```config_file
[app:service_v3]
use = egg:keystone#service_v3
```
而nova-api里用比较直接的方式引过去,我反正是不打算纠结了,知道怎么映射过去的就差不多.

这里也可以看出openstack的代码风格也不怎么统一
