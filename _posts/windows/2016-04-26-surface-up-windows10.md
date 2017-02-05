---
layout: post
title:  "surface pro3 升级win10的一些错误处理"
date:   2016-04-26 15:05:00 +0800
categories: "troubleshooting"
tag: ["windows"]
---

* content
{:toc}



错误日志

    Resource ‘$(string.RequirePrivateStoreOnly)’ referenced in attribute displayName could not be found
    File C:\WINDOWS\PolicyDefinitions\WindowsStore.admx, line 140, column 9

解决办法[参考](http://www.winhelponline.com/blog/gpedit-resource-string-requireprivatestoreonly-windowsstore-admx-kb3147458/)


先备份  C:\Windows\PolicyDefinitions\WindowsStore.admx,然后修改访问控制

    takeown /f C:\Windows\PolicyDefinitions\WindowsStore.admx
    icacls C:\Windows\PolicyDefinitions\WindowsStore.admx /grant Administrators:F

编辑C:\Windows\PolicyDefinitions\WindowsStore.admx   这是一个xml文件

    133-166行删除并保存

还原

    icacls C:\Windows\PolicyDefinitions\WindowsStore.admx /setowner "NT Service\TrustedInstaller"


surface pro3 升级后windows.old不能被磁盘清理掉,源自两个驱动

    SurfaceDisplayCalibration.sys
    SurfaceAccessoryDevice.sys

查看hardlink的命令

    fsutil hardlink list SurfaceAccessoryDevice.sys
    可以看到在windows文件中已经有这个文件,所以可以直接删除这两文件

猜测问题原因应该是这两个驱动hardlink到新的windows目录后没有删除旧目录中的hardlink文件就加载了驱动,因此无法被删除,
因此删除方法是先去除这两个文件的访问控制,然后卸载对应驱动后删除再这两文件

这两驱动位于设备管理器的系统设备中,对应名字分别为

    Surface Accessory Device
    Surface Display Calibration

## 卸载这两驱动（不要勾选删除驱动文件）

卸载后windows.old文件夹里的对应sys文件就可以删除了(直接删除不要丢回收站,不然回收站里也清不掉)

删除windows.old文件夹即可删除(注意修改下层文件夹权限与访问控制,cmd也要返回顶层)

## 我最近发现进恢复模式 rd c:\windows.old 最方便.....
