---
layout: post
title:  "OpenStack Mitaka从零开始 创建openstack所用镜像"
date:   2017-03-01 15:05:00 +0800
categories: "虚拟化"
tag: ["openstack", "linux"]
---

* content
{:toc}

最近才发现之前一直用的镜像是lvm的不支持rezise,没办法还得自己做,搞个openstack真TM什么都要自己来

linux镜像的制作过程参考[这篇](http://blog.csdn.net/xiegh2014/article/details/53248403)文章

以下操作在openstack已经搭建完成的情况下操作的,没搭建好的更加简单,直接virsh环境里就能处理了

---

#### 宿主机中操作

确保宿主机的evelent版本

    大于python2-eventlet-0.18.4-1

    默认版本python2-eventlet-0.17.4-4.el7监听unix socket有问题...


```shell
mkdir /data/tmp
# 系统硬盘
qemu-img create -f qcow2 centos-6.8.qcow2 5G
# swap分区专用硬盘,不然安装的时候会卡在selinux安装
qemu-img create -f raw swap.raw 4G
# 修改文件权限
chmod 777 /data/tmp/*
```


页面上关闭openstack虚拟机

重命名虚拟机instance-00xxxxx.xml文件为image.xml(xml文件可以不用备份,openstack每次启动都会覆盖)

```xml
<!--编辑name,愿意的话uuid也可以改掉-->
<name>image</name>

<!--删除以下内容-->
<metadata>
    ...
<metadata>

<console type='file'>
    <source path='/xxxx/xxxx/console.log'/>
    <target type='serial' port='0'/>
</console>

<serial type='file'>
  <source path='/xxxx/xxxx/console.log'/>
  <target port='0'/>
</serial>


<!--编辑下面file部分指向刚才创建的centos-6.8.qcow2-->
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2' cache='none'/>
  <source file='/data/tmp/centos-6.8.qcow2'/>
  <target dev='vda' bus='virtio'/>
  <address type='pci' domain='0x0000' bus='0x00' slot='0x04' function='0x0'/>
</disk>

<!--编辑下面boot以cdrom启动-->
<os>
  ....
  <boot dev='cdrom'/>
  ...
</os>


<!--在disk后面增加内容给你,file部分指向Centos的ios安装镜像-->
<disk type='file' device='cdrom'>
    <source file='/data/tmp/CentOS-6.8-x86_64-minimal.iso'/>
    <target dev='hdb' bus='ide'/>
</disk>
<!--指向swap.raw,没有swap安装会卡住很久-->
<disk type='file' device='disk'>
  <driver name='qemu' type='raw' cache='none'/>
  <source file='/data/tmp/swap.raw'/>
  <target dev='vdb' bus='virtio'/>
</disk>

<!--确保 虚拟机内存大于700M, centos必须启动内存要求大于628M的图形安装界面才能分区-->

```


启动虚拟机

```shell
virsh
    define image.xml
    start image
```
通过vnc安装系统

    系统盘只创建/boot和/分区
    挂载那个专门当swap的盘放swap

---

#### 虚拟机中操作

修改配置文件禁用selinux

修改/etc/fstab

    设置根分区noatime
    删除swap分区

修改/etc/ssh/sshd.conf,禁止DNS和GSS

chkconfig关闭多余的系统服务

关闭虚拟机

---

#### 宿主机中操作

虚拟机image  xml中的swap分区,cdrom信息删除,启动盘设置为硬盘

启动虚拟机

---

#### 虚拟机中操作

删除/etc/udev/rules.d/70-persistent-net.rules中当前网卡信息

修改网卡配置文件 /etc/sysconfig/network-scripts/ifcfg-eth0

     DEVICE=eth0
     ONBOOT=yes
     BOOTPROTO=dhcp
     NM_CONTROLLED=no

grub.conf的kernel参数增加如下内容(用于nova console-log获取),启动timeout改1

    console=tty0 console=ttyS0,115200n8

增加一行到/etc/sysconfig/network,这个设置避免和openstack的169地址冲突

    NOZEROCONF=yes

按照需要设置/etc/sysconfig/clock

    UTC=false

安装cloud init源,安装cloud init， resize等工具

```shell
# 安装并启动acpid,方便外部关闭电源
yum -y install acpid
chkconfig acpid on
# ntpdate没有什么依赖
# 还有lrzsz,zip，unzip,dos2unix,vim之类依赖低的都可以装上
# 自用镜像不用考虑极限压缩镜像大小
yum -y intall ntpdate
# cloud inti rpm包源
yum install -y http://dl.Fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
# 下载resize工具,不要搞什么git,直接下压缩包下来
wget https://codeload.github.com/flegmatik/linux-rootfs-resize/zip/master
# 安装cloud-init
yum install -y cloud-utils cloud-init parted
# 安装 resize
unzip linux-rootfs-resize.zip
cd linux-rootfs-resize
./install
# 删除下载包
yum clean ALL
cd ..
rm -rf linux-rootfs-resize.zip linux-rootfs-resize
```

修改cloud init配置文件

```yaml
# 这我也感觉是操作用户的,删了 >_,<
# users:
# - default

# 允许 root
disable_root: 1
# ssh允许账号密码登陆
ssh_pwauth:   0

# 不要管账户
# - users-groups
# 不要处理ssh
# - ssh
# 不要设置password
#  - set-passwords


# 不添加centos用户
# system_info:
#   default_user:
#     name: centos
#     lock_passwd: true
#     gecos: Cloud User
#     groups: [wheel, adm]
#     sudo: ["ALL=(ALL) NOPASSWD:ALL"]
#     shell: /bin/bash
#   distro: rhel
#   paths:
#     cloud_dir: /var/lib/cloud
#     templates_dir: /etc/cloud/templates
#   ssh_svcname: sshd

# 设置数据源
# 注意不要使用[]的方式来表示列表
# pytonh默认的yaml模块可能不识别
datasource:
  OpenStack:
    metadata_urls:
     - http://169.254.169.254
    max_wait: 10
    timeout: 3

datasource_list:
 - OpenStack

```

调用cloudinit来配置文件测试

```python
from cloudinit import safeyaml
import yaml
f = open('/etc/cloud/cloud.cfg','rb')
blob = f.read()
f.close()
try:
    converted = safeyaml.load(blob)
except (yaml.YAMLError, TypeError, ValueError), e:
    print str(e)
```

删除测试文件关机

---

#### 宿主机中操作

```shell
    # 安装libguestfs工具
    yum -y install libguestfs-tools
    # 清除镜像相关信息, -d 表示domain,所以这个命令在任意目录执行
    # loud init第一次执行的local模块错误警告可能是这个引起的,影响不大
    # virt-sysprep可以设置root密码,但是感觉还是被cloud init覆盖
    virt-sysprep -d image
    # 压缩虚拟机
    virt-sparsify --compress centos-6.8.qcow2 centos-6.8-image.qcow2
    # 接下来virsh undefined image,下载镜像并上传到glance
```

宿主机创建时执行脚本实例

```shell
#!/bin/bash
# 修改密码
echo "111111" | passwd root --stdin
# 遗漏的rpm包补充安装
rpm -i http://172.20.0.3/wproject/lrzsz-0.12.20-27.1.el6.x86_64.rpm
rpm -i http://172.20.0.3/wproject/ntpdate-4.2.6p5-10.el6.centos.2.x86_64.rpm
# 同步时间
/usr/sbin/ntpdate -s 172.20.0.3 && clock -w
```

---

#### windows 2008镜像之前也做好了,没做记录以后有空补上
