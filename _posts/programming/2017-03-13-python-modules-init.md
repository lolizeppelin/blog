---
layout: post
title:  "python 模块中的__init__"
date:   2017-03-13 12:50:00 +0800
categories: "编程"
tag: ["python", "linux"]
---

* content
{:toc}


这个问题源自于研究eventlet的时候,找不到neutron在哪调用hub.add,这样的话,neutron就是用monkey_patch来实现eventlet的，可是我没有在neutron里找到各个agent执行eventlet的monkey_patch代码


neutron的文档里有这么一段

    If a service utilizes the eventlet library, then it should not call
    eventlet.monkey_patch() directly but instead maintain its entry point main()
    function under neutron/cmd/eventlet/...

    If that is the case, the standard
    Python library will be automatically patched for the service on entry point
    import (monkey patching is done inside....

    正确使用eventlet,只要把agent的入口放到neutron/cmd/eventlet/下就会自动调用monkey_patch


我操....import的包内的其他东西会自动调用__init__的内容？

测试了一下

文件结构如下

```text
modinit
 - agent
    - __init__.py
    - l3.py
 - __init__.py
test.py
```

modinit.__init__.py

```python
print 'modinit init'
```

modinit.agent.__init__.py

```python
print 'agent init'
```

modinit.agent.l3.py

```python
def main():
    print 'main l3'
```
test.py
```python
print 'test.py'
from modinit.agent.l3 import main
main()
```

执行test.py的输出结果

    modinit init
    agent init
    test.py
    main l3


我了个擦今天才知道

一些其他相关知识[参考](http://www.cnblogs.com/tqsummer/archive/2011/01/24/1943273.html)
