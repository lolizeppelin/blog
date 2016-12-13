---
layout: post
title:  "OpenStack Mitaka从零开始 keystone配置文件"
date:   2016-12-06 15:05:00 +0800
categories: "虚拟化"
tag: ["openstack", "python"]
---

* content
{:toc}



|selection|key|默认值|可选取值|说明|
|:--|:--|:--|:--|:--|
|From keystone       keystone 基本配置| | | | |
|DEFAULT|admin_token|None| |管理员token,使用这个token的就是管理员,通过[filter:admin_token_auth]实现.新安装的时候数据库里没管理员,所以没法登陆,设置这个管理员token,那么以这个token登陆的都是管理员|
|DEFAULT|public_endpoint|http://server:5000| |比如你站点配置不在根目录,使用了下级目录的时候,需要配置这两个值指向下级目录.例如http://server:5000/path|
|DEFAULT|admin_endpoint|http://server:35357|同上|
|DEFAULT|max_project_tree_depth|5|int|项目深度,子项目下面还有子项目|
|DEFAULT|max_param_size|64|int|最大id长度|
|DEFAULT|max_token_size|8192|int|token最大长度|
|DEFAULT|member_role_id|9fe2ff9ee4384b1894a90878d3e92bab| |add_user_to_project函数会调用member_role_id,如果数据库里找不到对应的member_role_id会用配置文件里的id和name去创建一个.dashboard里会配置OPENSTACK_KEYSTONE_DEFAULT_ROLE需要与这个相同.member_role_id最好使用一个md5形式的字符串以便统一|
|DEFAULT|member_role_name|_member_| | |
|DEFAULT|crypt_strength|10000|1000~100000|加密字符串的长度,越长加密消耗的性能越多|
|DEFAULT|list_limit|None|int|全局list长度,默认为无限制,各个selection中的以自己的list_limit为准|
|DEFAULT|domain_id_immutable|True|true  false|是否允许切换domain  即将移除的配置|
|DEFAULT|strict_password_check|False|true  false|设置为false的时候,自动截断超过长度的password|
|DEFAULT|secure_proxy_ssl_header|HTTP_X_FORWARDED_PROTO| |ssl有关   估计对应的http服务里要带入的http head|
|DEFAULT|insecure_debug|False|true  false|和调试有关|
|From keystone.notifications     对外通知配置,比如配置notification_opt_out=identity.user.created.那么当创建用户的时候,就会发送一个对外通知.这个是走ampq的, publisher_id就是注册的publisher, 接收者要自己弄一个, 这个东西是用来做keystone审计用的| | | | |
|DEFAULT|default_publisher_id|socket.gethostname()| |默认为None, 代码中会自动使用socket.gethostname()|
|DEFAULT|notification_format|basic|basic, cadf|cadf是basic的扩展, 比basic多了发起者的信息|
|DEFAULT|notification_opt_out|[]|identity.user.created identity.authenticate.success|可以通过重复配置这个key来支持多个notification_opt_out,默认为空列也就是不对外发送消息|

---
    石帆胜丰
---



|From oslo.log   log配置相关      里面的配置说明比较清晰,就不再这里列出了| | | | |
|From oslo.messaging rpc通信的配置, 因为只使用rabbit,所以和rabbit没关系的都不列出,这里可以看出配置还是有点乱 为了兼容以前的还保留了好多字段, zmq的所有配置都还在default里,  后面oslo_messaging_rabbit部分还有大量rabbit配置| | | | |
|DEFAULT|rpc_backend|rabbit|amqp  zmq rabbit|rpc通信的方式, 看代码里好像还有fake kafka pika等方式,rabbit驱动就是对kombu的再次封装,其实都是pika的封装|
|DEFAULT|transport_url| | |不要配置这个, 这个直接在rabbit里面配置就好,之前应该是没有rpc_backend这个配置,直接通过transport_url解析出用那个backend和对应链接信息|
|DEFAULT|rpc_conn_pool_size|30|int|连接池大小,不建议在DEFAULT中配置（过时配置）,，建议配置到oslo_messaging_rabbit.rpc_conn_pool_size(用rabbit)|
|DEFAULT|rpc_response_timeout|60|int|rpc  send的的timeout|
|DEFAULT|executor_thread_pool_size|64|int|取代[DEFAULT]/rpc_thread_pool_size,keystone的executor用,默认也就是eventlet的工作"线程"数|
|DEFAULT|control_exchange|keystone| |amqp的交换机名字|
|From oslo.service.service    守护进程相关配置| | | | |
|DEFAULT|backdoor_port|0| |为0随机端口,没有IP配置是因为之监听本地IP|
|DEFAULT|backdoor_socket| | |用unix sock方式   这个和backdoor_port应该是二选一的,应该是用于通知keystone进程关闭重启之类的后门操作的接口,实际测试建议不要unix sock,eventlet有代码print出host和端口,用这个日志里会有一条错误记录|
|DEFAULT|log_options|True|true  false|是否记录日志|
|DEFAULT|graceful_shutdown_timeout|60|int|关闭超时时间|
|From keystone   [assignment]   keystone.assignment.backends.sql:Assignment   __tablename__ = 'assignment', 这个就是给role分配权限的和处理user和role关系的,里面有诸如add_grant  delete_grant  add_role_to_user_and_project 之类的方法| | | | |
|assignment|driver|sql| |只支持sql, 不指定的话将使用[identity]/driver中的配置,从代码里看,不配置的时候从identity里强制返回sql,而且代码中有注明,今后使用非sql的时候必须指定而不能从identity中继承|
|assignment|prohibited_implied_role|admin| |禁止implied的角色,这个具体看下面翻译文档|
|From keystone    [auth]   key的auth配置   这个应该对应v3 的 auth| | | | |
|auth|methods|external,password,token,oauth1| |允许使用的验证方式|
|auth|password|Password| |keystone.auth.password中的namespace（也就是指向的类名,不用设置）|
|auth|token|Token| |keystone.auth.token中的namespace（也就是指向的类名,不用设置）|
|auth|external|DefaultDomain| |keystone.auth.external中的namespace（也就是指向的类名,不用设置）|
|auth|oauth1|Oauth1| |keystone.auth.oauth1中的namespace（也就是指向的类名,不用设置）|
|From oslo.cache   [cache]   全局缓存配置, 这里的memcache服务器与[memcache]中的memcache区别是,这里的memcache是作为cache的memcache,所有的只要启用了cache就会用上这个,[memcache]的memcache是作为store的的, 其实目前也就只有token用memcache作store而已,token使用memcache作为backend的情况下,token就不需要再启用cache了| | | | |
|cache|config_prefix|cache.oslo| |缓存key的前缀|
|cache|expiration_time|600| |超时时间|
|cache|backend|dogpile.cache.null|dogpile.cache.memcache,oslo_cache.memcache_pool|缓存驱动,默认的null不知道是什么缓存方式|
|cache|backend_argument| | |驱动的参数,我们用oslo_cache.memcache_pool不需要参数|
|cache|proxies| | |dogpile.cache的配置,我们用oslo_cache.memcache_pool可以不用管|
|cache|enabled|False|true  false|是否启用全局缓存, 若全局缓存没启用,其他selection里的cache启用了也无效|
|cache|debug_cache_backend|False|true  false|缓存 get  set delete的日志开关|
|cache|memcache_servers|localhost:11211| |memcache服务器连接信息 backend配置为memcache相关的时候才有效.注意,这里和后面[memcache]中的memcache服务器是没关系的.不过这两个地方配置为同一个memcahe也没问题,因为都有自己的prefix. oslo_cache.memcache_pool实际使用的时候会把参数前面的memcache去掉, 比如memcache_pool_maxsize在oslo_cache.memcache_pool中是pool_maxsize, memcache_socket_timeout用于soeckt.settimeout()|
|cache|memcache_dead_retry|300| | |
|cache|memcache_socket_timeout|3| | |
|cache|memcache_pool_maxsize|10| |oslo_cache.memcache_pool专有参数, 连接池大小|
|cache|memcache_pool_unused_timeout|60| |oslo_cache.memcache_pool专有参数, 空闲链接时间|
|cache|memcache_pool_connection_get_timeout|10| |从memcache的pool中获取一个链接的超时时间, 这个时间和socket无关,最终是queue的get超时时间,也就是说超时socket的相关超时时间由memcache_socket_timeout控制|
|From keystone  [catalog]| | | | |
|catalog|template_file|default_catalog.templates| |模板文件位置,当driver配置为templated时这个参数才有效|
|catalog|driver|sql|kvs, sql,  templated, endpoint_filter.sql|驱动方式|
|catalog|caching|True|true  false|是否使用缓存,如果使用kvs和templated估计可以不用cache|
|catalog|list_limit| | |返回列表长度限制|
|From oslo.middleware   [cors]   COR就是跨域资源共享, 这里的配置自然就是处理跨域的,和http服务器的跨域配置是一个东西,没有需求可以不配置,这个是filter,而且是第一个filter, 跨域的内容可以参考http://netsecurity.51cto.com/art/201311/419179.htm, w3c文档https://www.w3.org/TR/cors/#access-control-allow-methods-response-header| | | | |
|cors|allowed_origin| | |对应nginx配置Access-Control-Allow-Origin   允许的跨域的url|
|cors|allow_credentials|True|true  false|对应nginx配置Access-Control-Allow-Credentials   是否允许请求带有验证信息|
|cors|expose_headers|X-Auth-Token,X-Openstack-Request-Id X-Subject-Token| |对应nginx配置Access-Control-Expose-Headers   允许脚本访问的返回头，请求成功后，脚本可以在XMLHttpRequest中访问这些头的信息|
|cors|max_age|3600| |对应nginx配置Access-Control-Max-Age  缓存此次请求的秒数|
|cors|allow_methods|GET,PUT,POST,DELETE,PATCH| |对应nginx配置Access-Control-Allow-Methods   允许的跨域的methods|
|cors|allow_headers|X-Auth-Token,X-Openstack-Request-Id X-Subject-Token,X-Project-Id X-Project-Name,X-Project-Domain-Id X-Project-Domain-Name X-Domain-Id,X-Domain-Name| |对应nginx配置r Access-Control-Allow-Headers  允许自定义的头部|
|From oslo.middleware   [cors.subdomain]  这里的配置和上面是一样的,这里的subdomain是你自己的域名,比如aaa.com, 配置头就写cors.aaa.com, 指定的域配置覆盖上层配置, 代码里专门看了下,"cors."后面的的字符串可以随便写,代码中实际使用的是的allowed_origin,"cors."后面配置得和allowed_origin一样是为了方便看而已| | | | |
|cors.subdomain|allowed_origin|与cors配置相同| | |
|cors.subdomain|allow_credentials| | | |
|cors.subdomain|expose_headers| | | |
|cors.subdomain|max_age| | | |
|cors.subdomain|allow_methods| | | |
|cors.subdomain|allow_headers| | | |
|From keystone   [credential]   只支持sql, 这玩意我现在还没看明白是干嘛的, 好像也是v2的时候才有的东西,现在都直接在auth_routers通过authenticate_for_token登陆了,好像都用不到credential了| | | | |
|credential|driver|sql| | |
|From oslo.db   [database]   数据库连接信息| | | | |
|database|sqlite_db| | |用mysql就不用管这个|
|database|sqlite_synchronous| | |用mysql就不用管这个|
|database|db_backend|sqlalchemy| |sqlalchemy是通用的数据库框架   这个配置取代了[DEFAULT]/db_backend|
|database|connection| | |取代[DEFAULT]/sql_connection  [DATABASE]/sql_connection   [sql]/connection|
|database|slave_connection| | |以前没有的 新加的从库连接, 现在支持读写分离？|
|database|mysql_sql_mode|TRADITIONAL| |mysql独有配置|
|database|sql_idle_timeout|3600| |取代[DEFAULT]/sql_idle_timeout  [DATABASE]/sql_idle_timeout [sql]/idle_timeout|
|database|min_pool_size|1| |取代[DEFAULT]/sql_min_pool_size  [DATABASE]/sql_min_pool_size|
|database|max_pool_size| | |取代[DEFAULT]/sql_max_pool_size  [DATABASE]/sql_max_pool_size|
|database|max_retries|10| | |
|database|retry_interval|10| | |
|database|max_overflow|50| | |
|database|connection_debug|0|0=None, 100=Everything| |
|database|connection_trace|False|true  false| |
|database|pool_timeout| | | |
|database|use_db_reconnect|False|true  false| |
|database|db_retry_interval|1|int|Seconds between retries of a database transaction.应该是语句执行出错重试,前面的重试是链接不通的重试|
|database|db_inc_retry_interval|True|true  false|是否增长重试时间,每重试出错一次,重试间隔时间加1, 下面的参数就是增加到的间隔上限|
|database|db_max_retry_interval|10|int|重试间隔时间上限|
|database|db_max_retries|20| |重试到达这个次数就报错   设置-1不报错|
|From keystone  [domain_config] | | | | |
|domain_config|driver|sql| | |
|domain_config|caching|True|true  false| |
|domain_config|cache_time|300|int| |
|From keystone   [endpoint_filter] | | | | |
|endpoint_filter|driver|sql| |sql指向keystone.catalog.backends.sql:Catalog, 这个和catalog用的一个驱动|
|endpoint_filter|return_all_endpoints_if_no_filter|True|true  false|代码里只有一个地方用了这个参数,估计也是兼容v2用的|
|From keystone   [endpoint_policy]   获取终点的policy| | | | |
|endpoint_policy|enabled|True|true  false|这个参数已经过时,下个版本会默认启用不再允许禁止, M版中,启用endpoint_policy会增加一个routes|
|endpoint_policy|driver|sql| | |
|From keystone [eventlet_server]     keystone守护进程相关配置, 以eventlet_server（也就是通过keystone-all）来启动keystone已经是不推荐的设置,所以这里的配置都可以不关注了, 现在建议直接使用uwsgi的方式启动keystone,具体看uwsgi启动keystone的部分| | | | |
|eventlet_server|public_workers|cpu 个数|int|公共api app的进程数量|
|eventlet_server|admin_workers|cpu 个数|int|admin api app的进程数量|
|eventlet_server|public_bind_host|0.0.0.0| |取代[DEFAULT]/bind_host   [DEFAULT]/public_bind_host|
|eventlet_server|public_port|5000| |取代[DEFAULT]/public_port |
|eventlet_server|admin_bind_host|0.0.0.0| | |
|eventlet_server|admin_port|35357| | |
|eventlet_server|wsgi_keep_alive|True|true  false|http 1.1支持的http长连接是否开启, 也就是respoen后是否close socket|
|eventlet_server|client_socket_timeout|900|int|取0 永不超时, 代码里是accetp到socket后调用socket.settimeout|
|eventlet_server|tcp_keepalive|False|int|取代[DEFAULT]/tcp_keepalive,代码里accpect后设置socket.SO_KEEPALIVE,设置这个就表示操作系统来检查连接是否已经断开(心跳)|
|eventlet_server|tcp_keepidle| | |设置了上面参数后,设置设个参数做心跳检查间隔|
|From keystone [eventlet_server_ssl]   暂时不启用ssl,先不理会这里的配置| | | | |
|From keystone [federation] 联合身份认证,现在用不上,不用配置,用于外部认证用户,这类用户信息不存放在keystone中, 联合的api将外部user添加一个映射,并将用户映射到一个Keystone组. 参考http://www.ibm.com/developerworks/cn/cloud/library/cl-keystone-tfim/| | | | |
|From keystone   [fernet_tokens]   这个是对应[token]里的provider配置  一般用uuid 或者pki、pkiz,所以不关注这个配置先. 看文档这个tokens方式的性能比uuid和pik都要好好很多| | | | |
|From keystone [identity]    __tablename__ = 'user'    auth中authenticate是通过identity_api来验证用户的,identity就处理用户信息的. user 的get  delete  update add之类都是通过identity| | | | |
|identity|default_domain_id|default| |v2 没有domain的概念, 给v2 api指定一个domain|
|identity|domain_specific_drivers_enabled|False|true  false|子domain或者全部domain的配置通过resource的backend获取   false  (新版推荐设置)子domain或者全部domain的配置通过下面domain_configurations_from_database指定true. -------------------参考1:  https://www.ibm.com/developerworks/cn/cloud/library/cl-configure-keystone-ldap-and-active-directory/  -------参考2:  http://docs.openstack.org/developer/keystone/configuration.html Setting domain_specific_drivers_enabled to True will enable this feature, causing Keystone to look in the domain_config_dir for config files of the form:keystone.<domain_name>.conf|
|identity|domain_configurations_from_database|False|true  false|domain配置通过resource的backend中获取true,domain配置从文件读取false.也就是说在domain_specific_drivers_enabled为true的情况下,要使用resource的backend, 需要手动指定为true.新版建议配置domain_configurations_from_database为false,也就是让这个配置无效|
|identity|domain_config_dir|/etc/keystone/domains| |当domain_specific_drivers_enabled设置为true且domain_configurations_from_database为false的时候,获取子domain配置文件的文件夹.也就是说当domain_specific_drivers_enabled为false或者domain_specific_drivers_enabled/domain_configurations_from_database双true的话这里就无效|
|identity|driver|sql| |identity 的 backend|
|identity|caching|True|true  false| |
|identity|cache_time|600|int| |
|identity|max_password_length|4096| |最长用户密码|
|identity|list_limit| | | |
|From keystone  identity_mapping  [identity_mapping]   也是兼容用的| | | | |
|identity_mapping|driver|sql| | |
|identity_mapping|generator|sha256| | |
|identity_mapping|backward_compatible_ids|False|true  false|整个keystone数据是新的建议设置为false,有老数据设置为true|
|From keystone   [kvs]  cache的配置,如果有使用kvs这里就必须配置.可以使用的kvs的有 catalog  token revoke,  kvs在token中还不可以禁止revoke_by_id. cache都使用memcache,所以这里的配置也不看了| | | | |
|From keystone  [ldap]   不使用ldap认真,不看配置先| | | | |
|From oslo.messaging  [matchmaker_redis]  使用zmq的时候要用到redis,我们使用rabbit不用管这里的配置| | | | |
|From keystone [memcache]   用到memcache的只有token(注意和cache里的memcache区别)| | | | |
|memcache|servers|localhost:11211| | |
|memcache|dead_retry|300| | |
|memcache|socket_timeout|3| | |
|memcache|pool_maxsize|10| | |
|memcache|pool_unused_timeout|60| | |
|memcache|pool_connection_get_timeout|10| | |
|From keystone   [oauth1]| | | | |
|oauth1|driver|sql| | |
|oauth1|request_token_duration|28800| | |
|oauth1|access_token_duration|86400| | |
|From keystone  [os_inherit]  用户继承| | | | |
|os_inherit|enabled|True|true  false|即将过时参数,今后会默认启用|
|From keystone   [oslo_messaging_amqp]  我们使用rabbit,这里的配置不用看| | | | |
|From oslo.messaging  [oslo_messaging_notifications]| | | | |
|oslo_messaging_notifications|driver| |messaging, messagingv2,routing, log, test, noop|通过配置多次支持多个driver, 不使用messaging/messagingv2  transport_url和topics可以不用配置|
|oslo_messaging_notifications|transport_url| | |不配置这个的话,使用默认的rpc配置（就是使用keystone的rpc）|
|oslo_messaging_notifications|topics|notifications| |是个列表|
|From oslo.messaging  [oslo_messaging_rabbit]   rabbit配置| | | | |
|oslo_messaging_rabbit|amqp_durable_queues|False|true  false|取代[DEFAULT]/amqp_durable_queues  [DEFAULT]/rabbit_durable_queues ampq里的持久化queue, 就是Consumer注册的时候的durable设置|
|oslo_messaging_rabbit|amqp_auto_delete|False|true  false|取代[DEFAULT]/amqp_auto_delete 是否自动删除exchange|
|oslo_messaging_rabbit|kombu_ssl_version| | |取代[DEFAULT]/kombu_ssl_version 不用ssl  可以不用配置|
|oslo_messaging_rabbit|kombu_ssl_keyfile| | |取代[DEFAULT]/kombu_ssl_keyfile|
|oslo_messaging_rabbit|kombu_ssl_certfile| | |取代[DEFAULT]/kombu_ssl_certfile|
|oslo_messaging_rabbit|kombu_ssl_ca_certs| | |取代[DEFAULT]/kombu_ssl_ca_certs|
|oslo_messaging_rabbit|kombu_reconnect_delay|1|float|取代[DEFAULT]/kombu_reconnect_delay. 重连延迟|
|oslo_messaging_rabbit|kombu_compression| |gzip,  bz2|是否启用数据压缩,默认不启用|
|oslo_messaging_rabbit|kombu_missing_consumer_retry_timeout|60|int|取代[DEFAULT]/kombu_reconnect_timeout,必须比rpc_response_timeout长, consumer重连重试时间|
|oslo_messaging_rabbit|kombu_failover_strategy|round-robin|round-robin, shuffle|failover方式|
|oslo_messaging_rabbit|rabbit_host|localhost| |取代[DEFAULT]/rabbit_host, rabbit 的host,这个配置代码中部使用,只有配置文件中将他和port合并为rabbit_hosts|
|oslo_messaging_rabbit|rabbit_port|5672| |取代[DEFAULT]/rabbit_port 端口,下面的配了这也可以不配|
|oslo_messaging_rabbit|rabbit_hosts|$rabbit_host:$rabbit_port| |取代[DEFAULT]/rabbit_hosts 可以配置为列表|
|oslo_messaging_rabbit|rabbit_use_ssl|False|true  false|取代[DEFAULT]/rabbit_use_ssl|
|oslo_messaging_rabbit|rabbit_userid|guest| |取代[DEFAULT]/rabbit_userid rabbit登陆账号|
|oslo_messaging_rabbit|rabbit_password|guest| |取代[DEFAULT]/rabbit_password|
|oslo_messaging_rabbit|rabbit_login_method|AMQPLAIN| |取代[DEFAULT]/rabbit_login_method|
|oslo_messaging_rabbit|rabbit_virtual_host|/| |取代[DEFAULT]/rabbit_virtual_host 自己建过专用的vhost的话前面不需要带上/  例如/openstack就是错的|
|oslo_messaging_rabbit|rabbit_retry_interval| | | |
|oslo_messaging_rabbit|rabbit_retry_backoff| | | |
|oslo_messaging_rabbit|rabbit_interval_max| | | |
|oslo_messaging_rabbit|rabbit_max_retries| | | |
|oslo_messaging_rabbit|rabbit_ha_queues| | | |
|oslo_messaging_rabbit|rabbit_transient_queues_ttl| | | |
|oslo_messaging_rabbit|rabbit_qos_prefetch_count|0| |rabbit的prefetch_count 设置为0就是无限制|
|oslo_messaging_rabbit|heartbeat_timeout_threshold|60| | |
|oslo_messaging_rabbit|heartbeat_rate|2| | |
|oslo_messaging_rabbit|fake_rabbit|False|true  false| |
|oslo_messaging_rabbit|channel_max| | | |
|oslo_messaging_rabbit|frame_max| | |ampq的最大帧字节限制|
|oslo_messaging_rabbit|heartbeat_interval|1| | |
|oslo_messaging_rabbit|ssl| | | |
|oslo_messaging_rabbit|ssl_options| | | |
|oslo_messaging_rabbit|socket_timeout|0.25|float| |
|oslo_messaging_rabbit|tcp_user_timeout|0.25|float| |
|oslo_messaging_rabbit|host_connection_reconnect_delay|0.25|float| |
|oslo_messaging_rabbit|pool_max_size|10|int| |
|oslo_messaging_rabbit|pool_max_overflow|0|int| |
|oslo_messaging_rabbit|pool_timeout|30|int| |
|oslo_messaging_rabbit|pool_recycle|600|int| |
|oslo_messaging_rabbit|pool_stale|60|int| |
|oslo_messaging_rabbit|notification_persistence|False|true  false| |
|oslo_messaging_rabbit|default_notification_exchange|${control_exchange}_notification| | |
|oslo_messaging_rabbit|notification_listener_prefetch_count|100| | |
|oslo_messaging_rabbit|default_notification_retry_attempts|-1| | -1 means infinite retry|
|oslo_messaging_rabbit|notification_retry_delay|0.25| | |
|oslo_messaging_rabbit|rpc_queue_expiration|60| | |
|oslo_messaging_rabbit|rpc_reply_exchange|${control_exchange}_rpc_reply| | |
|oslo_messaging_rabbit|rpc_listener_prefetch_count|100| | |
|oslo_messaging_rabbit|rpc_reply_listener_prefetch_count|100| | |
|oslo_messaging_rabbit|rpc_reply_retry_attempts|-1| |-1 means infinite retry during rpc_timeout|
|oslo_messaging_rabbit|rpc_reply_retry_delay|0.25|float| |
|oslo_messaging_rabbit|default_rpc_retry_attempts|-1| | -1 means infinite retry|
|oslo_messaging_rabbit|rpc_retry_delay|0.25|float| |
|From oslo.middleware   [oslo_middleware]  一些通用配置| | | | |
|oslo_middleware|max_request_body_size|114688|int|wsgi的最大请求大小,后端通过content_length来比较|
|oslo_middleware|secure_proxy_ssl_header|X-Forwarded-Proto| |ssl   proxy头,不用ssl不用管|
|From oslo.policy  [oslo_policy]| | | | |
|oslo_policy|policy_file| | |取代[DEFAULT]/policy_file|
|oslo_policy|policy_default_rule| | |取代[DEFAULT]/policy_default_rule|
|oslo_policy|policy_dirs| | |取代[DEFAULT]/policy_dirs|
|From keystone   [paste_deploy]   wsgi服务器启动的时候 urlmap映射调用的配置| | | | |
|paste_deploy|config_file|keystone-paste.ini| |paste deploy的配置文件位置|
|From keystone  [policy]| | | | |
|policy|driver|sql| | |
|policy|list_limit| | | |


---
    的打法
---
|From keystone  [resource]   resource就是操作project的(list  add delete update 之类), 从配置文件的取代可以看出和assignment(也就是role)的关系, project通过role来检查权限.代码里通过装饰器@controller.filterprotected 和@controller.protected来验证policy. 参考http://www.tuicool.com/articles/i2qUNf protected/filterprotected 最终会调用 /keystone/openstack/common/policy.py 中的 enforce 方法 keystone 与 openstack 其它component 中的 RBAC 实现的不同之处在于 keystone 中有时候需要做 token 验证再做 policy 检查。 Volume/Backup/ConsistencyGroup 等 API 类中的接口函数上的 @wrap_check_policy 或者在函数体内显式调用 check_policy(context, 'get', volume) 来判断调用用户的 role 是否满足 policy.json 中定义的要求。比如 /volume/api.py 中的 delete 方法：| | | | |
|resource|driver| | |如果这里不配置,则继承[assignment]/driver|
|resource|caching| | |取代[assignment]/caching|
|resource|cache_time| | |取代[assignment]/cache_time|
|resource|list_limit| | |取代[assignment]/list_limit|
|resource|admin_project_domain_name| | |在keystone里相关代码只有下面部分 if (project['name'] == r.admin_project_name and project['domain']['name'] == r.admin_project_domain_name):  token_data['is_admin_project'] = True|
|resource|admin_project_name| | | |
|resource|project_name_url_safe|off|off, new, strict|strict  严格检查url,不允许url中包含特定字符. new  create的时候相当于strict, 不是create的时候就相当于off. off  不检查url的的特殊字符. 因为仪表盘还有老接口使用,这里配置为off|
|resource|domain_name_url_safe|off|off, new, strict| |
|From keystone  [revoke]   revoke我的理解是, 因为受控是延迟收到命令的,执行前还要先去请求确认当前指令是否已经revoke(撤销)或者过时| | | | |
|revoke|driver| | | |
|revoke|expiration_buffer|1800| |通过当前时间减去token的超时时间加上这个expiration_buffer计算得到oldest. 在外部访问list_revoke_events的时候把所有revoked_at时间小于oldest的记录都删除. 确保要超过revoked_at时间点  expiration_buffer + tonek expiration那么多的时间才删除|
|revoke|caching| | | |
|revoke|cache_time| | |取代[token]/revocation_cache_time|
|From keystone  [role]  sql=keystone.assignment.role_backends.sql:Role assignment 这里是对role进行add  get  delete  update操作的, get的是后用户id可以通过group映射到多个role, role再通过policy检查对应权限| | | | |
|role|driver|sql| |目前只支持sql|
|role|caching|True|true  false| |
|role|cache_time| | | |
|role|list_limit| | | |
|From keystone   [saml]  这玩意看代码好像就只是和federation联合授权相关 暂时不看| | | | |
|From keystone   [shadow_users]   这玩意看代码好像也是和federation联合授权相关 暂时不看| | | | |
|From keystone   [signing]   ssl相关 暂时不看| | | | |
|From keystone   [ssl]   ssl相关 暂时不看| | | | |
|From keystone  [token]  token相关| | | | |
|token|bind| | |这两都是扩展认证的, bind里只看见了对kerberos执行了追加验证, 估计其他的追加认证要自己对authenticate函数做修改|
|token|enforce_token_bind|permissive|disabled, permissive, strict| |
|token|expiration|3600|int| |
|token|provider|uuid|fernet,pkiz,pki,uuid|fernet是最新的,性能好像最好|
|token|driver|sql|kvs, memcache, memcache_pool, sql|我们这里直接用memcached_pool|
|token|caching|True| |我们用了memcached,这里就可以关了,当然想做二级缓存也可以 |
|token|cache_time| | | |
|token|revoke_by_id|True|true  false|是否允许有token认证的identifier撤销这个token, 除非使用kvs存储,否则建议设置为true|
|token|allow_rescope_scoped_token|True|true  false|为true的话,允许调用authenticate的时候,token中包含有project和domain的scope|
|token|hash_algorithm|md5| |PKI tokens用的参数,即将过时参数,不用管|
|token|infer_roles|True|true  false|看说明感觉是临时性将role链接到其他role, 这样就不用显示的增加role, implied_role相关,参考后面implied_role的部分|
