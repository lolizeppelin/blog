---
layout: post
title:  "一次瞎了眼的配置导致的dnsmasq调试"
date:   2017-01-21 15:05:00 +0800
categories: "debug"
tag: ["python", "linux"]
---

* content
{:toc}


dns日志

    no address range available for DHCP request via eth3

网上搜索回复都是子网设置问题,看了半天配置文件没看出问题,配置文件改来改去也没发解决,最后只能硬着头皮读dnsmasq的源码然后调试了

配置1

    # eth0 对应172.20.0.0/23网段
    # interface=eth0
    # listen-address=172.20.0.3
    # dhcp-range=vlan20,172.20.1.5,172.20.1.125,255.255.254.0,1h
    # dhcp-range=vlan20,172.20.1.191,172.20.1.254,255.255.254.0,1h
    # ------------------
    dhcp-range=vlan20,172.20.1.5,172.20.1.125
    dhcp-range=vlan20,172.20.1.191,172.20.1.254
    dhcp-option=net:vlan20,3,172.20.0.1  # 网关
    dhcp-option=net:vlan20,6,172.20.0.3  # dns服务器
    dhcp-option=net:vlan20,19,0          # 用禁止转发,加快windows服务器自动分配
    dhcp-option=net:vlan20,66,172.20.0.3 # tfptserver服务器位置
    dhcp-boot=net:vlan20,pxelinux.0


配置2

    # eth3 对应172.16.0.0/24网段
    # interface=eth3
    # listen-address=172.21.0.254
    # dhcp-range=vlan21,172.21.1.10,172.21.1.250,255.255.255.0,1h
    dhcp-range=vlan21,172.21.0.10,172.21.0.250
    dhcp-option=net:vlan21,3,172.21.0.1  # 网关
    dhcp-option=net:vlan21,6,172.20.0.3  # dns服务器
    dhcp-option=net:vlan21,19,0


对应日志位置

```c++
if (!context)
  {
    my_syslog(LOG_WARNING, _("no address range available for DHCP request %s %s"),
      subnet_addr.s_addr ? _("with subnet selector") : _("via"),
      subnet_addr.s_addr ? inet_ntoa(subnet_addr) : (mess->giaddr.s_addr ? inet_ntoa(mess->giaddr) : iface_name));
    return 0;
  }
```

debug

断点位置

    (gdb) stop  # 暂停进程
    (gdb) break complete_context if if_index==5  # 遇到eth3网开始断点
    (gdb) continue # 继续进程
    # 在for (context = daemon->dhcp; context; context = context->next)循环
        (gdb) print is_same_net(local, context->start, netmask)
        $18 = 0
        (gdb) print local
        $19 = {s_addr = 4261418412}
        (gdb) print context->start
        $20 = {s_addr = 167843244}
        (gdb) print netmask
        $21 = {s_addr = 16777215}
        (gdb) print context->end
        $22 = {s_addr = 4194375084}

最终结果是local(eth3上的IP)和子网开始结束不匹配
认真看清楚了子网范围是172.21.1.10,172.21.1.250
而本地网卡的ip是 172.20.0.254, 第三位是0, 子网的是1

卧槽...原来真是日志所反映的子网配置错误-_-
