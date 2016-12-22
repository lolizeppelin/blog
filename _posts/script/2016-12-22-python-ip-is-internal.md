---
layout: post
title:  "判断IP地址是否为内网IP"
date:   2016-12-14 12:50:00 +0800
categories: "工具脚本"
tag: ["python"]
---

* content
{:toc}


[参考地址](http://www.codeweblog.com/python%E5%88%A4%E6%96%AD%E5%86%85%E7%BD%91ip/)


```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-

def ip_into_int(ip):
    # 先把 192.168.1.13 变成16进制的 c0.a8.01.0d ，再去了“.”后转成10进制的 3232235789 即可。
    # (((((192 * 256) + 168) * 256) + 1) * 256) + 13
    return reduce(lambda x,y:(x<<8)+y,map(int,ip.split('.')))

def is_internal_ip(ip):
    ip = ip_into_int(ip)
    net_a = ip_into_int('10.255.255.255') >> 24
    net_b = ip_into_int('172.31.255.255') >> 20
    net_c = ip_into_int('192.168.255.255') >> 16
    return ip >> 24 == net_a or ip >>20 == net_b or ip >> 16 == net_c

if __name__ == '__main__':
    ip = '192.168.0.1'
    print ip, is_internal_ip(ip)
    ip = '10.2.0.1'
    print ip, is_internal_ip(ip)
    ip = '172.16.1.1'
    print ip, is_internal_ip(ip)
```

稍微改一下

```python
#!/usr/bin/python
# -*- coding: UTF-8 -*-

import re

# 10.255.255.255
NET_A = 10
# 172.31.255.255'
NET_B = 2753
# 192.168.255.255
NET_C = 49320

IP_RE = r"\b(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4]" \
        r"[0-9]|[01]?[0-9][0-9]?)\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\." \
        r"(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)\b"

def is_ipv4addr(var):
    """
    判断是否为参数ip
    """
    if not isinstance(var, (str, unicode)):
        return False
    if isinstance(var, unicode):
        try:
            var = var.encode('utf-8')
        except (TypeError, ValueError, AttributeError):
            return False
    if len(var) > 15:
        return False
    if re.match(IP_RE, var):
        return True
    else:
        return False

def is_internal_ipv4(ip_addr):
    if not is_ipv4addr(ip_addr):
        return False
    # 先把 192.168.1.13 变成16进制的 c0.a8.01.0d ，再去了“.”后转成10进制的 3232235789 即可。
    # (((((192 * 256) + 168) * 256) + 1) * 256) + 13
    int_ip = reduce(lambda x,y:(x<<8)+y,map(int, ip_addr.split('.')))
    return int_ip >> 24 == NET_A or int_ip >>20 == NET_B or int_ip >> 16 == NET_C

```
