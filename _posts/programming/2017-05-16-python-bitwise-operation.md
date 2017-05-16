---
layout: post
title:  "python 位运算Bitwise operation"
date:   2017-05-16 12:50:00 +0800
categories: "编程"
tag: ["python", "linux"]
---

* content
{:toc}


之前模拟Snowflake做了个主键生成的方法,为了避免位运算用直接48位时间,8为server,8位序列
然后用struct打包

最近用起来发现没有线程/进程ID还是不行,256个serverid也不够用

还是用位操作照着Snowflake来做比较合适

问题就出来了,所有位运算的简介都没提到python的里位移后长度怎么算的......折腾了一个下午终于搞明白了

    << 方式位移后长度 将是 当前值的bit长度加上位移长度
    在编译语言里当前值的长度是定义好的
    python当前值的位长度在位运算里这样计算
    1是1bit 8是3bit......当然也可以理解为前面有足够多的为0的bit
    动态语言真是卧槽.........

    如果你想生成指定长度的bit需要先算一下当前值的字节数再位移


    不同长度的数进行 | & 运算的时候, 短长度的会自动在前面补0变得一样长
    也就是说不需要刻意移动到足够长度再移回来


下面我们来一个模拟的全局主键生成,生成的主键和Snowflake一样,正好是个8字节的long long

```python
class Gprimarykey(object):
    """A global primark key maker like Snowflake
    0-42    time                   42  bit
    42-53   sid       max 2047     11  bit
    53-61   pid       max 255      8   bit
    61-64   sequence  max 7        3   bit
    """
    def __init__(self, sid, pid,
                 diff=int(time.time()*1000) - int(monotonic()*1000),
                 ):
        self.__diff = diff
        self.__last = 0
        self.__sequence = 0
        if pid >=256 or sid >= 2048:
            raise RuntimeError('sid should less then 2048 pid should less then 256')
        self.__sid = sid
        self.__pid = pid

    def update_diff(self, diff):
        self.__diff = diff

    def __call__(self):
        return self.makekey(self.__sid, self.__pid)

    def makekey(self, sid, pid):
        """Make a global primark key"""
        if pid >=256 or sid >= 2048:
            raise RuntimeError('sid should less then 2048 pid should less then 256')
        cur = int(monotonic()*1000) + self.__diff
        if self.__last == cur:
            if self.__sequence >= 8:
                time.sleep(0.001)
                # recursive call
                return self.makekey(sid, pid)
            self.__sequence += 1
        else:
            self.__sequence = 0
            self.__last = cur
            # 当前毫秒时间左位移22位
            # 当前时间如果超过
            # 42 bit的最大值是4398046511103
            # 也就是2109-05-15 15:35:11
            # 这个类返回的数值就会大过long long的范围
            # 也就是超过了64 bit
            # 如果要完整模拟Snowflake
            # 也就是41位时间 1位0
            # 就要先要算出41bit的最大值然后用
            # 位操作&处理当前时间
            # 参考sidformat中的处理方式
            part_time = cur << 22
            part_server = sid << 11
            part_pid = pid << 3
            # 合并每个位置对应的bit
            key = part_time | part_server |  part_pid | self.__sequence
            return key

    def timeformat(self, key):
        return key >> 22

    def sidformat(self, key):
        # 先用server 的最大值 2047 位移到server id所在位置
        # 2047是11bit的的最大值, 加上11bit就正好到达server的bit位置
        # 和key做 与运算
        # 向右位移到开头(删除后面的0)
        return (key & (2047 << 11)) >> 11
```
