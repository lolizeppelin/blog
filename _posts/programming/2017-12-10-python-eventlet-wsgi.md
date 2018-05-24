---
layout: post
title:  "python eventlet http长连接, keepalive, close wait"
date:   2017-12-10 12:50:00 +0800
categories: "编程"
tag: ["python", "linux"]
---

* content
{:toc}


直接说重点

    eventlet中的socket timeout是假的,不是操作系统告诉你的

    是eventlet计时（epoll监控的fd一直没返回）到点后抛出的

    如果python版本为2.6触发一个bug导致socket不能关闭

    所以即使底层用了tcp keeplive保证了tcp,但是wsgi服务器上处理当前socket的绿色线程已经因为socket timout消失了

    当后续包发到服务器后,由于处理当前socket数据的线程已经没了,自然就没有数据响应

    在抓包结果中看到的就是 web服务器收到了完整的http请求包,但是只有tcp层ack没有没有http回复

    客户端会因为超时而关闭当前长连接


### 核心重点1：eventlet中的socket timeout是HUB抛出的！！！

### 核心重点2：python 2.6的BaseRequestHandler的写法会出发bug不能关闭socket

修复方法很简单, 继承BaseHTTPRequestHandler类用python2.7的init方法替换就是

bug产生的原因如下

    eventlet的HttpProtocol(继承自python的BaseHTTPRequestHandler)在接收数据的时候为了方便
    通过dup把socket复制成了file对象
    由于python2.6的BaseRequestHandler没有在finally中调用finish
    导致异常发生的时候没有关闭掉dup出来的fd
    所以调用了socket.close并不能关闭socket
