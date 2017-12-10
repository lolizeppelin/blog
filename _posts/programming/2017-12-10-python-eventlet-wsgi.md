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

    即使底层用了tcp keeplive保证了tcp,但是wsgi服务器上处理当前socket的绿色线程已经socket timout了

    当后续包发到服务器后,由于处理当前socket数据的线程已经没了,自然就没有数据响应

    在抓包结果中看到的就是 web服务器收到了完整的http请求包,但是只有tcp层ack没有没有http回复

    客户端会因为超时而关闭当前长连接

    要想长连接不断,则需要在wsgi服务器socket timeout的时间内发一个完整的http请求（类似自写心跳包）

    如果请求不密集,写http心跳包有点脱裤子放屁, 如果请求密集,自然不需要http心跳包


### 核心重点就是eventlet中的socket timeout是HUB抛出的！！！

### 客户端必须实现心跳,底层keepalive完全无效,因为eventlet不知道
