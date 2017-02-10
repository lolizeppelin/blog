---
layout: post
title:  "Linux ip命令的使用以及路由基础知识"
date:   2017-01-16 15:05:00 +0800
categories: "网络基础"
tag: ["linux"]
---

* content
{:toc}




路由选择 [文档地址](http://linux-ip.net/html/routing-selection.html#routing-selection-adv)


    Crucial to the proper ability of hosts to exchange IP packets is the correct
    selection of a route to the destination. The rules for the selection of route path are
    traditionally made on a hop-by-hop basis [18] based solely upon the destination address of
    the packet. Linux behaves as a conventional routing device in this way,
    but can also provide a more flexible capability. Routes can be chosen and
    prioritized based on other packet characteristics.
    主机实现交换IP包的关键点在于能正确的选择到达目的地的路由，通常的选择路由路径的规则是在
    hop-by-hop协议基础上,通过网络包的目的地址来制定的,linux支持常规的方式的同时
    还提供了更加灵活的扩展功能,也就是说linux支持通过网络包的其他属性来路由

    The route selection algorithm under linux has been generalized to enable
    the powerful latter scenario without complicating the overwhelmingly common case of the former scenario.
    除非复杂度远超普通情形, 否则linux下的路由选择算发(已经被认可)在常见情况下远远好于以往的方式

    The above sections on routing to a local network and the default gateway
    expose the importance of destination address for route selection.
    In this simplified model, the kernel need only know the destination address of the packet,
    which it compares against the routing tables to determine the route by which to send the packet.
    上面提到的本地和默认网关的路由选择可以看出目的地址对路由选择的重要性
    在这种简单模式种,那可只需要知道目的地的地址,然后去路由表中匹配即可确定网络包的所需路由

    The kernel searches for a matching entry for the destination first in the routing cache
    and then the main routing table. In the case that the machine has recently transmitted
    a packet to the destination address, the routing cache will contain an entry for the destination.
    The kernel will select the same route, and transmit the packet accordingly.
    内核首先在路由缓存中搜索匹配的目标条目，然后搜索主路由表(local表)。
    在机器最近向相同目的地址发送过数据包的情况下，路由缓存中会包含匹配目的地址的的路由条目。
    内核将选择缓存中返回的路由（不去查询rule、和route table），并相应地传输网络包。

    If the linux machine has not recently transmitted a packet to this destination address,
    it will look up the destination in its routing table using a technique known longest prefix match [19].
    In practical terms, the concept of longest prefix match means that the most specific route to the destination will be chosen.
    如果机器最没有向相同目的地址发过数据包(没有路由缓存),内核将会在路由表中查找目的地址相同的掩码最长的条目
    实际上,匹配到最长掩码就意味着选择到目的地的具体路由





```c++
if packet.routeCacheLookupKey in routeCache :
    route = routeCache[ packet.routeCacheLookupKey ]
else
    for rule in rpdb :
        if packet.rpdbLookupKey in rule :
            routeTable = rule[ lookupTable ]
            if packet.routeLookupKey in routeTable :
                route = route_table[ packet.routeLookup_key ]
```


    This pseudocode provides some explanation of the decisions required to find a route.
    The final piece of information required to understand the decision
    making process is the lookup process for each of the three hash table lookups.
    In Table 4.1, “Keys used for hash table lookups during route selection”,
    each key is listed in order of importance.
    Optional keys are listed in italics and represent keys that will be matched if they are present.



### Table 4.1. Keys used for hash table lookups during route selection

| route cache| RPDB        |route table(kernel)|
| -----------|:-----------:|:-----------:|
| destination| source      | destination |
| source     | destination |   ToS       |
| ToS        | ToS         |    scope    |
| fwmark     | fwmark      |    oif      |
| iif        | iif         |      null   |


---

ip link[文档地址](http://linux-ip.net/html/tools-ip-link.html)


    The ip link tool provides the following two verbs: ip link show and ip link set.

    B.3.1. Displaying link layer characteristics with ip link show
    通过ip link show显示链路层(OSI 二层, 802.2、802.3ATM、HDLC、FRAME RELAY)属性

    To display link layer information, ip link show will fetch characteristics of the link layer devices currently available.
    Any networking device which has a driver loaded can be classified as an available device.
    It is immaterial to ip link whether the device is in use by any higher layer protocols (e.g., IP).
    You can specify which device you want to know more about with the dev <interface> option.
    为了显示链路层信息,  ip link show会刷新当前可达的链路层设备的属性
    所有加载了驱动的网络设备都被认为是可达的设备
    对于ip link来说,设备是否已经被高层的协议(例如 IP协议)所使用是无关紧要的
    你可以指定通过指定dev <interface>参数的方式来获取你想要的设备的更多信息


---

scope是什么 相关[文档地址](http://linux-ip.net/html/tools-ip-address.html#tb-tools-ip-addr-scope)


    Scope is normally determined by the ip utility without explicit use on the command line.
    For example, an IP address in the 127.0.0.0/8 range falls in the range of localhost IPs,
    so should not be routed out any device. This explains the presence of
    the host scope for addresses bound to interface lo. Usually,
    addresses on other interfaces are public interfaces, which means
    that their scope will be global. We will revisit scope again
    when we discuss routing with ip route, and there we will also encounter the link scope.
    scope通常在ip的命令中不被显式的指定,例如,127.0.0.0/8 ip段中的IP都是本机ip。
    因此不需要路由到任何设备,local表中的那条scope host到lo接口的那条记录就是一个列子
    通常来说, 地址所在的其他接口(lo以外接口)的是公共接口,也就是说他们的scope是全局的.当我们使用ip route讨论路由时
    我们将再次使用到scope,那时候我们还会遇到并讨论link scope


    man 8 ip
    scope SCOPE_VAL
           the scope of the destinations covered by the route prefix.  SCOPE_VAL may be a number or a  string  from
           the  file  /etc/iproute2/rt_scopes.  If this parameter is omitted, ip assumes scope global for all gate-
           wayed unicast routes, scope link for direct unicast and  broadcast  routes  and  scope  host  for  local
           routes.
           被路由前缀(子网掩码)覆盖的目的地的scope, SCOPE_VAL可以是/etc/iproute2/rt_scopes包含的字符串
           如果这个参数没有被指定, ip(ip这个cmd)将假定作用范围是所有设置了gatewary的单播路由
           scope link 用于确定的单播和多播路由
           scope host 用于本地(回环)路由


    [root@second ~]# cat /etc/iproute2/rt_scopes
    0       global
    255     nowhere
    254     host
    253     link
    #
    # pseudo-reserved
    #
    200     site

scop的常用类型

| Scope  | Description                          |
|--------|:------------------------------------:|
| global | valid everywhere                     |   全局有效
| site   | valid only within this site (IPv6)   |   IPV6 地址有效
| link   | valid only on this device            |   当前设备有效
| host   | valid only inside this host (machine)|   当前机器有效(这个一般出现在local路由表里)

---

路由表

    ip route - routing table management
               # 路由表管理

           Manipulate route entries in the kernel routing tables keep information about paths to other networked nodes.
           操作 (保存了联通其他网络节点路径的信息的)kernel路由表 中的(具体)路由条目.(keep前有一个which比较好理解)

           Route types:

                   unicast - the route entry describes real paths to the destinations covered by the route prefix.
                   单播       路由条目描述了到达目的地的真实路径,目的地址被路由前缀(掩码)覆盖过

                   unreachable - these destinations are unreachable. Packets are  discarded  and  the  ICMP  message  host
                   unreachable is generated.  The local senders get an EHOSTUNREACH error.

                   blackhole  - these destinations are unreachable. Packets are discarded silently.  The local senders get
                   an EINVAL error.

                   prohibit - these destinations are unreachable. Packets are discarded and the ICMP message communication
                   administratively prohibited is generated. The local senders get an EACCES error.

                   local  - the destinations are assigned to this host. The packets are looped back and delivered locally.

                   broadcast - the destinations are broadcast addresses. The packets are sent as link broadcasts.

                   throw - a special control route used together with policy rules. If such a route is selected, lookup in
                   this table is terminated pretending that no route was found. Without policy routing it is equivalent to
                   the absence of the route in the routing table. The  packets  are  dropped  and  the  ICMP  message  net
                   unreachable is generated. The local senders get an ENETUNREACH error.
                   返回给rule处理

                   nat  - a special NAT route. Destinations covered by the prefix are considered to be dummy (or external)
                   addresses which require translation to real (or internal) ones  before  forwarding.  The  addresses  to
                   translate  to  are selected with the attribute via.  Warning: Route NAT is no longer supported in Linux
                   2.6.

                   anycast - not implemented the destinations are anycast addresses assigned to this host. They are mainly
                   equivalent  to local with one difference: such addresses are invalid when used as the source address of
                   any packet.

                   multicast - a special type used for multicast routing. It is not present in normal routing tables.


          Route tables: Linux-2.x can pack routes into several routing tables identified by a number in the range from  1
          to 255 or by name from the file /etc/iproute2/rt_tables By default all normal routes are inserted into the main
          table (ID 254) and the kernel only uses this table when calculating routes.

          Actually, one other table always exists, which is invisible but even more important. It is the local table  (ID
          255). This table consists of routes for local and broadcast addresses. The kernel maintains this table automat-
          ically and the administrator usually need not modify it or even look at it.

          The multiple routing tables enter the game when policy routing is used.
          当策略路由使用的时候,多个路由表将加入"游戏",具体使用那个路由表,参考后面的ip rule
          在没有设置过ip rule的情况下,所有网络包都先走一遍local表,在local表被没匹配就走main表(在走local表之前会先查询路由缓存)
          我们直接运行route和ip route显示的就是main表的内容
          default路由表是一个路由表


          [root@second ~]# cat /etc/iproute2/rt_tables
          255     local
          254     main
          253     default
          0       unspec


proto是什么

    protocol RTPROTO
           the  routing  protocol  identifier  of  this  route.   
           RTPROTO may be a number or a string from the file /etc/iproute2/rt_protos.  
           If the routing protocol ID is not given, ip assumes  protocol  boot  
           (i.e.  it assumes  the  route  was  added by someone
           who doesn’t understand what they are doing).
           Several protocol values have a fixed interpretation.  Namely:
           路由的协议识标符，RTPROTO可能是一个数字或者string,具体查询 /etc/iproute2/rt_protos
           如果协议ID没有指定,ip命令将协议默认为boot(协议为boot的路由被认为是无效路由)
           一些常用的已知路由协议


                   redirect - the route was installed due to an ICMP redirect.
                              用于IMCP重定向的路由

                   kernel - the route was installed by the kernel during autoconfiguration.
                            由内核自动设置的路由

                   boot - the route was installed during the bootup sequence.  
                   If a routing daemon starts, it  will purge all of them.
                          这个路由在启动序列的时候被生成,当一个路由daemon启动后,将清理掉protocol为boot的路由
                          前面有说明,没有指定protocol的路由将会默认为是boot

                   static  - the route was installed by the administrator to override dynamic routing.
                   Routing daemon will respect them and, probably, even advertise them to its peers.
                             由管理员生成的,覆盖动态路由,路由进程将按照其中的设置通知到其他同伴

                   ra - the route was installed by Router Discovery protocol.
                        由路由发现协议生成的路由

           The rest of the values are not reserved and the administrator
           is free to assign (or not to assign)  protocol tags.
           剩下的数值都不是保留协议值,管理员可以任意处理


           [root@second ~]# cat /etc/iproute2/rt_protos
           #
           # Reserved protocols.
           #
           0       unspec
           1       redirect
           2       kernel
           3       boot
           4       static
           8       gated
           9       ra
           10      mrt
           11      zebra
           12      bird
           13      dnrouted
           14      xorp
           15      ntk
           16      dhcp



ip rule是什么


    ip rule - routing policy database management
              路由策略数据库管理

           Rules in the routing policy database control the route selection algorithm.
           路由策略数据库中的规则控制路由选择算法

           Classic routing algorithms used in the Internet make routing decisions based only on the destination address of
           packets (and in theory, but not in practice, on the TOS field).
           在互联网中使用的经典路由算法只通过网络包的目的地地址来生成路由策略（仅仅在理论上，而实践中，在TOS设置的情况下不是）。

           In  some  circumstances  we  want to route packets differently depending not only on destination addresses, but
           also on other packet fields: source address, IP protocol, transport protocol  ports  or  even  packet  payload.
           This task is called ’policy routing’.
           在某些情况下，我们想要不同地路由分组，不仅取决于目的地地址，
           而且取决于网络包的其他属性,例如：源地址，IP协议，传输协议端口或甚至网络包的有效载荷。 这被称为“策略路由”。

           To  solve  this  task, the conventional destination based routing table, ordered according to the longest match
           rule, is replaced with a ’routing policy database’ (or RPDB), which selects routes by  executing  some  set  of
           rules.
           为了实现策略路由，根据最长匹配规则排序的常规的基于目的地的路由表被替换为“路由策略数据库”（或RPDB），其通过执行一些规则集来选择路由。


           Each  policy  routing  rule  consists  of  a selector and an action predicate.  The RPDB is scanned in order of
           decreasing priority. The selector of each rule is applied to {source  address,  destination  address,  incoming
           interface,  tos, fwmark} and, if the selector matches the packet, the action is performed. The action predicate
           may return with success.  In this case, it will either give a route or failure indication and the  RPDB  lookup
           is terminated. Otherwise, the RPDB program continues with the next rule.
           每条策略路由规则包含了两个对象, 匹配对象、动作行为
           RPDB按优先级递减的顺序扫描(优先级0最高,32767最低)
           每条规则的匹配对象类型为{原地址、目标地址、传入的接口、tos标记、fwmar标记}
           当网络包被匹配对象匹配后,动作将被执行
           动作行为可能返回成功,在这种情况下,RPDB将返回一个路由或者失败指示,RPDB同时查询终止
           如果动作行为没有返回成功,RPDB进入下一条rule匹配
           (也就是说Priority数值较低的会被优先执行,高优先规则动作执行成功后不会匹配到后面的规则,动作执行失败进入下一条规则)。


           Semantically, the natural action is to select the nexthop and the output device.
           在字面意义上，一般的的动作是选择下一跳和输出设备。

           At startup time the kernel configures the default RPDB consisting of three rules:
           内核启动时设置的默认路由策略管理数据库包含3条规则


           1.     Priority:  0, Selector: match anything, Action: lookup routing table local (ID 255).  The local table is
                  a special routing table containing high priority control routes for local and broadcast addresses.
                  优先级0, 最高优先级, 匹配对象:所有  动作：查询路由表local（路由表ID 255）
                  local路由表是一个特别的路由表,他包含有控制本地地址和广播地址的路由控制,优先级高

                  Rule 0 is special. It cannot be deleted or overridden.
                  优先级为0规则是特殊的,他不能被删除或者覆盖

                  也就是说用户空间不能操作local表, broadcast（广播）和local(本地回环)都是用户设置IP的时候由内核来修改local表


           2.     Priority: 32766, Selector: match anything, Action: lookup routing table main (ID 254).  The  main  table
                  is the normal routing table containing all non-policy routes. This rule may be deleted and/or overridden
                  with other ones by the administrator.

                  优先级32766, 匹配对象:所有  动作：查询路由表main（路由表ID 254）
                  main路由表是个普通的路由表,包含了所有没有策略规则的路由, 这条规则可以通过管理员删除或覆盖


           3.     Priority: 32767, Selector: match anything, Action: lookup routing table default (ID 253).   The  default
                  table  is  empty.  It  is  reserved  for  some post-processing if no previous default rules selected the
                  packet.  This rule may also be deleted.

                  优先级32767, 匹配对象:所有  动作：查询路由表default（路由表ID 255）
                  default路由表默认为空


           Each RPDB entry has additional attributes. F.e. each rule has a pointer to some routing  table.  NAT  and  mas-
           querading  rules  have  an attribute to select new IP address to translate/masquerade. Besides that, rules have
           some optional attributes, which routes have, namely realms.  These values do not override  those  contained  in
           the routing tables. They are only used if the route did not select any attributes.
           每个RPDB条目额外的属性, 例如, 每条规则都会指向一些对应的路由表
           nat和mas-querading（伪装）规则包含一个新的用于转义或伪装的IP地址
           除此以外,规则中还有一些路由的可选属性,也就是realms(the realm to which this route is assigned路由被指派的作用域,看下面额外说明)
           规则中的这些值不会覆盖路由表中的相同值, 这些值只会在路由没有选择任何属性的时候被使用


           realms, 文档说明http://www.policyrouting.org/iproute2.doc.html#ss9.9
           参考说明http://superuser.com/questions/193561/what-does-the-content-of-etc-iproute2-rt-realms-mean-ubuntu-10-4

           Realms in iproute2 are a way of clustering sets of routes into groups.
           Packets following each route will be considered part of the corresponding realm,
           this classification can then be used to do bandwidth throttling, apply filters (e.g. with iptables),
           track usage statistics, and more, realm-wise.

           iproute2中的Realms是将路由集合到组中的一种方法。
           这个组中每个路由中的网络包被当作相应组的一部分来对待
           通过这种方式可以对同时多个路由进行流控、过滤、监控等(对组进行流控就等于对组中所有路由做流控)


           The RPDB may contain rules of the following types:
           策略路由数据库包含以下规则类型(ip rule可处理route的类型少于route table的route类型?)
           参考http://linux-ip.net/html/routing-rpdb.html

                   unicast - the rule prescribes to return the route found in the routing table referenced by the rule.
                   单播   规则将返回一条在路由表中被这条规则引用的路由
                   blackhole - the rule prescribes to silently drop the packet.
                   黑洞  规则用于silently丢弃网络包
                   unreachable - the rule prescribes to generate a ’Network is unreachable’ error.
                   不可达   规则生成一个网络不可达的错误
                   prohibit - the rule prescribes to generate ’Communication is administratively prohibited’ error.
                   禁止  规则用于生成一个“通信被禁止”的错误
                   nat - the rule prescribes to translate the source address of the IP packet into some other value.
                   nat   规则用于将网络包的原地址转义为其他指定地址


机器上ip rule查询

    [root@second ~]# ip rule show
    0:      from all lookup local
    32766:  from all lookup main
    32767:  from all lookup default

ip rule使用

   ip rule [ list | add | del ] SELECTOR ACTION

   SELECTOR := [ from PREFIX ] [ to PREFIX ] [ tos TOS ] [ fwmark FWMARK ] [ dev STRING ] [ pref NUMBER ]

   ACTION := [ table TABLE_ID ] [ nat ADDRESS ] [ prohibit | reject | unreachable ] [ realms [SRCREALM/]DSTREALM ]

   TABLE_ID := [ local | main | default | NUMBER ]


---

分割线,虚拟化好像没用到ip rule,都是默认规则

也就是所有网络包都走一遍local表,没匹配就走main表

---












重新认识NAT

    Full network address translation, as performed with iproute2 can be
    simulated with both netfilter SNAT and DNAT,
    with the potential benefit (and attendent resource consumption) of connection tracking
    完整的网络地址翻译，可以通过netfilter的SNAT和DNAT来模拟iproute2
    这种方式更方便做网络追踪,但同时也增加一些资源消耗


    NAT introduces a complexity to the network in which
    it is used because a service is reachable on a public and a private IP.
    NAT使得同时用于联通公共IP和私有IP的网络更为复杂


rule中nat和route table中nat的区别,[文档地址](http://linux-ip.net/html/nat-stateless.html#ftn.idm140337856992368)




路由规则例子(ucloud云主机)

    # local路由表
    [root@localhost ~]# ip route show table local
    broadcast 10.13.255.255 dev eth0  proto kernel  scope link  src 10.13.72.2
    broadcast 127.255.255.255 dev lo  proto kernel  scope link  src 127.0.0.1
    broadcast 127.0.0.0 dev lo  proto kernel  scope link  src 127.0.0.1
    broadcast 10.13.0.0 dev eth0  proto kernel  scope link  src 10.13.72.2
    local 10.13.72.2 dev eth0  proto kernel  scope host  src 10.13.72.2
    local 127.0.0.1 dev lo  proto kernel  scope host  src 127.0.0.1
    local 127.0.0.0/8 dev lo  proto kernel  scope host  src 127.0.0.1
    # main路由表
    [root@dyb-mszl185-Center ~]# ip route show table main
    10.13.0.0/16 dev eth0  proto kernel  scope link  src 10.13.72.2
    default via 10.13.0.1 dev eth0



    [root@openstack ~]# ip netns exec snat-0c7df318-411b-4172-92a2-0227c6d85584 ip route
    default via 172.30.0.1 dev qg-7c9d90f1-b0
    172.30.0.0/24 dev qg-7c9d90f1-b0  proto kernel  scope link  src 172.30.0.2
    172.30.1.0/24 dev qg-7c9d90f1-b0  scope link
