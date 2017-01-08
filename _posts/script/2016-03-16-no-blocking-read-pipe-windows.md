---
layout: post
title:  "windows非阻塞读取管道内容 "
date:   2016-12-14 12:50:00 +0800
categories: "工具脚本"
tag: ["python", "windows"]
---

* content
{:toc}



linux下很容易实现的方法在windows下行不通....windows下select不能操作管道,百度找了半天蛋都碎了,最后只能祭出google

### 感谢github！！！！看到github上的内容的时候,真是太开心了

[参考来源](https://gist.github.com/techtonik/48c2561f38f729a15b7b
)

[windows对应pipe的API](https://msdn.microsoft.com/en-us/library/aa365781%28v=vs.85%29.aspx)

举一反三,弄了两种写法,第一种和github的一样.第二种写法用了PeekNamedPipe.
还有一种写法可以用TransactNamedPipe,但是TransactNamedPipe要用到的一个结构体ctype.wintypes里没有封装,所以就算了.
顺便linux写法也丢上去了.


```python
# -*- coding: UTF-8 -*-
__author__ = 'loliz_000'


import msvcrt
import platform
import time
# 这个自己写的一个错误类,不用纠结
from _exception import CmdException


def read_line_thread_unix(fileobj, switch):
    import select
    while 1:
        # switc用来控制这个读取线程退出
        if switch['switch']:
            break
        try:
            read_list, write_list, error_list = select.select([fileobj], [], [], 0.01)
        except OSError:
            break
        except IOError:
            break
        except select.error:
            break
        if len(read_list) > 0:
            output = read_list[0].readline()
            if not output:
                break
            print output
            switch['out_line_list'].append(output)


def read_line_thread_windows1(fileobj, switch):
    import time
    from ctypes import windll, byref, GetLastError, WinError
    # 将指定类型转化为参数类型的函数
    from ctypes.wintypes import POINTER
    # 载入windows handle类型
    from ctypes.wintypes import HANDLE as HANDLE_TYPE
    # 载入windows dword类型
    from ctypes.wintypes import DWORD as DWORD_TYPE
    # 载入window是bool类型
    from ctypes.wintypes import INT as INT_TYPE
    # DWORD类型指正
    LPDWORD_TYPE = POINTER(DWORD_TYPE)
    # pipe nowait参数, dword 类
    PIPE_NOWAIT = DWORD_TYPE(0x00000001)
    # pipe 为空时的返回码
    ERROR_NO_DATA = 232
    # 定位SetNamedPipeHandleState函数
    SetNamedPipeHandleState = windll.kernel32.SetNamedPipeHandleState
    # 确定SetNamedPipeHandleState的参数类型
    SetNamedPipeHandleState.argtypes = [HANDLE_TYPE, LPDWORD_TYPE, LPDWORD_TYPE, LPDWORD_TYPE]
    # 确定SetNamedPipeHandleState的返回值类型
    SetNamedPipeHandleState.restype = INT_TYPE
    # 管道的handle值
    handle = msvcrt.get_osfhandle(fileobj.fileno())
    # 设置管道非阻塞
    res = SetNamedPipeHandleState(handle, byref(PIPE_NOWAIT), None, None)
    # 设置结果
    if res == 0:
        raise CmdException('set no block pipe read catch error %s' % str(WinError()))
    # 开始从管道读取数据
    while 1:
        if switch['switch']:
            break
        try:
            string_buff = fileobj.readline()
        except OSError:
            if GetLastError() != ERROR_NO_DATA:
                break
            else:
                raise CmdException('read from pipe catch error %s' % str(WinError()))
        if len(string_buff) == 0:
            time.sleep(0.001)
            continue
        else:
            print string_buff


def read_line_thread_windows2(fileobj, switch):
    from ctypes import windll, byref, GetLastError, WinError
    # 将指定类型转化为参数类型的函数
    from ctypes.wintypes import POINTER
    # 载入windows handle类型
    from ctypes.wintypes import HANDLE as HANDLE_TYPE
    # 载入windows dword类型
    from ctypes.wintypes import DWORD as DWORD_TYPE
    # 载入window是bool类型
    from ctypes.wintypes import INT as INT_TYPE
    # LPVOID_TYPE指针
    from ctypes.wintypes import LPVOID as LPVOID_TYPE
    # DWORD类型指正
    LPDWORD_TYPE = POINTER(DWORD_TYPE)
    # 期望读取长度
    BUFFER_SHOULD_BE_RES = DWORD_TYPE(4096)
    # 实际读取长度
    BUFFER_SIZE_RES = DWORD_TYPE(0)
    # 实际发送长度
    BUFFER_SIZE_SEND = DWORD_TYPE(0)
    # pipe 为空时的返回码
    ERROR_NO_DATA = 232
    # 定位PeekNamedPipe函数
    PeekNamedPipe = windll.kernel32.PeekNamedPipe
    # 确定PeekNamedPipe的参数类型
    PeekNamedPipe.argtypes = [HANDLE_TYPE, LPVOID_TYPE, DWORD_TYPE, LPDWORD_TYPE, LPDWORD_TYPE, LPDWORD_TYPE]
    # 确定PeekNamedPipe的返回值类型
    PeekNamedPipe.restype = INT_TYPE
    # 管道的handle值
    handle = msvcrt.get_osfhandle(fileobj.fileno())
    # 开始从管道读取数据
    while 1:
        if switch['switch']:
            break
        # 获取管道信息
        res = PeekNamedPipe(handle, None, BUFFER_SHOULD_BE_RES,
                            byref(BUFFER_SIZE_RES), byref(BUFFER_SIZE_SEND), None)
        # 设置结果
        if res == 0:
            print 'get end!!'
            break
            #raise CmdException('set no block pipe read catch error %s' % str(WinError()))
        if BUFFER_SIZE_RES.value <= 0:
            time.sleep(0.001)
            continue
        else:
            try:
                gg = fileobj.read(BUFFER_SIZE_RES.value)
                print gg
            except OSError:
                if GetLastError() != ERROR_NO_DATA:
                    break
                else:
                    raise CmdException('read from pipe catch error %s' % str(WinError()))
        #print gg.decode('gbk')
        #print 'finsh'


if platform.system() == 'windows':
    read_line_thread = read_line_thread_unix
else:
    read_line_thread = read_line_thread_windows1
```
