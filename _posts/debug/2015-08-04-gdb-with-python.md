---
layout: post
title:  "python配合gdb调试程序"
date:   2015-08-04 15:05:00 +0800
categories: "debug"
tag: ["python"]
---

* content
{:toc}



### 注意是gdb通过python脚本来调试程序,不是调试python代码,和pdb不是一回事,可以理解为gdb进程里实现了python虚拟机


[参考1](http://segmentfault.com/q/1010000000154532)

    如果觉得--ex很多行很麻烦 可以只--ex "source xxx.py" 来跑一个PYTHON脚本帮你


[参考2](http://stackoverflow.com/questions/4792483/how-to-import-gdb-in-python)

[详细API](https://sourceware.org/gdb/current/onlinedocs/gdb/Python-API.html#Python-API)


### 错误使用方式

    [root@second ~]# python
    Python 2.6.6 (r266:84292, Jan 22 2014, 09:42:36)
    [GCC 4.4.7 20120313 (Red Hat 4.4.7-4)] on linux2
    Type "help", "copyright", "credits" or "license" for more information.
    >>> import gdb
    Traceback (most recent call last):
    File "<stdin>", line 1, in <module>
    ImportError: No module named gdb
    >>>


### 正确方式

首先,创建一个用于测试脚本

```python
import gdb
help(gdb)
```

测试效果

    [root@second ~]# gdb -q -x 1.py
    Help on package gdb:
    NAME
    gdb
    FILE
    (built-in)
    PACKAGE CONTENTS
    FrameIterator
    FrameWrapper
    backtrace
    command (package)
    function (package)
    CLASSES
    __builtin__.object
    Block
    BlockIterator
    Breakpoint
    Command
    Field
    Frame
    Function
    Inferior
    InferiorThread
    Membuf
    Objfile
    Parameter
    Progspace
    Symbol
    Symtab
    Symtab_and_line
    Type
    Value
    exceptions.Exception(exceptions.BaseException)
    GdbError
    class Block(__builtin__.object)
