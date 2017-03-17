---
layout: post
title:  "drbd部分配置参数说明"
date:   2016-11-15 15:05:00 +0800
categories: "troubleshooting"
tag: ["drbd", "集群"]
---


* content
{:toc}


官方文档[地址](https://www.drbd.org/en/doc/users-guide-84/re-drbdconf)

有些参数要去Working with DRBD里找详细说明,有些参数去Optimizing DRBD performance找详细说明,反正旁边一堆文档....

```text
    local-io-error "/usr/lib/drbd/notify-io-error.sh; /usr/lib/drbd/notify-emergency-shutdown.sh; echo o > /proc/sysrq-trigger";
    --on-no-data-accessible ond-policy
    This setting controls what happens to IO requests on a degraded, disk less node (I.e. no data store is reachable).
    The available policies are io-error and suspend-io.

    当读写请求出现degraded的时候,控制器对错误的处理方式(可选方式为io-error和suspend-io)
    degraded这里指的是raid的一种状态,默认当作io-error异常处理, node里的配置覆盖disk中配置

```


    disk-barrier在 linux-2.6.36 (or 2.6.32 RHEL6)以后不安全,默认no
    disk-flushes是默认值,默认yes, 有raid电池可以关闭提高性能
    disk-drain这个只在8.0.9前用
    no-disk-drain 这个文档说不要用

---

    These options affect write performance on the secondary node.
    max-buffers is the maximum number of buffers DRBD allocates for
    writing data to disk while max-epoch-size is the maximum number of
    write requests permitted between two write barriers. max-buffers
    must be equal or bigger to max-epoch-size to increase performance.
    The default for both is 2048; setting it to around 8000 should be fine
    for most reasonably high-performance hardware RAID controllers.

    max-buffers  drbd每次从磁盘申请的预写入空间大小
    max-epoch-size 两次写入之间内存中最多保存多少数据

---

    unplug-watermark
    When the number of pending write requests on the standby (secondary) node exceeds the unplug-watermark,
    Some controllers perform best when "kicked" frequently, so for these
    controllers it makes sense to set this fairly low, perhaps even as low as
    DRBD’s allowable minimum (16). Others perform best when left alone; for
    these controllers a setting as high as max-buffers is advisable.

    当standby (secondary)节点的未写入数据达到unplug-watermark值,将节点踢掉
    有些控制器平凡的踢掉性能会更好,这种时候使用最小值16都是可以的
    其他的控制器更倾向于不理会,这种情况况下可以设置得和max-buffers值一样大

---

    cram-hmac-alg
    Configure the hash-based message authentication code (HMAC) or secure hash
    algorithm to use for peer authentication. The kernel supports a number of
    different algorithms, some of which may be loadable as kernel modules.
    See the shash algorithms listed in /proc/crypto. By default, cram-hmac-alg
    is unset. Peer authentication also requires a shared-secret to be configured.
    hash-based message authentication code  基于散列的消息认证码
    secure hash algorithm to use for peer authentication.   对等认证的安全散列算法
    不启用不校验,cpu占用低延迟也低一点


---

看activity log相关配置前先来看看一个关于metadata的说明
```text
metadata
DRBD将数据的各种信息块保存在一个专用的区域里，这些metadata包括了
a，DRBD设备的大小
b，产生的标识
c，活动日志(这里不是指上面的activity log,而是指活动、非活动区域的移入移除信息)
d，快速同步的位图
metadata的存储方式有内部和外部两种方式，使用哪种配置都是在资源配置中定义的
内部meta data
内部metadata存放在同一块硬盘或分区的最后的位置上
优点：metadata和数据是紧密联系在一起的，如果硬盘损坏，metadata同样就没有了，同样在恢复的时候，metadata也会一起被恢复回来
缺点：metadata和数据在同一块硬盘上，对于写操作的吞吐量会带来负面的影响，因为应用程序的写请求会触发metadata的更新，这样写操作就会造成两次额外的磁头读写移动。
外部meta data
外部的metadata存放在和数据磁盘分开的独立的块设备上
优点：对于一些写操作可以对一些潜在的行为提供一些改进
缺点：metadata和数据不是联系在一起的，所以如果数据盘出现故障，在更换新盘的时候就需要认为的干预操作来进行现有node对心硬盘的同步了
如果硬盘上有数据，并且硬盘或者分区不支持扩展，或者现有的文件系统不支持shrinking，那就必须使用外部metadata这种方式了。
可以通过下面的命令来计算metadata需要占用的扇区数
```

接下来是activity log的配置

    al-extents extents
    DRBD automatically maintains a "hot" or "active" disk area likely to be
    written to again soon based on the recent write activity. The "active"
    disk area can be written to immediately, while "inactive" disk areas must
    be "activated" first, which requires a meta-data write. We also refer
    to this active disk area as the "activity log".
    drdb自动维护一个磁盘可能马上要被写入的热点(活动)区域,
    热点(活动)区域是一个可以马上被写入的区域
    非活动的区域需要先被激活才能被写入数据,而这个激活的过程会产生一个meta-data写请求
    我们将这个热点/活动区域称为activity log
    activity log区域太小的话,会因为空间不足导致不停将数据块区移入、移出activity log区域
    这就会平凡的写meta-data,造成性能下降

    The activity log saves meta-data writes, but the whole log must be resynced
    upon recovery of a failed node. The size of the activity log is a major
    factor of how long a resync will take and how fast a replicated disk will
    become consistent after a crash.
    activity log保存了meta-data写入的数据,当恢复一个失败节点的时候
    activity log必须被完整的重新同步,activity log的大小决定了恢复失败节点所需要的时间

    The activity log consists of a number of 4-Megabyte segments; the al-extents
    parameter determines how many of those segments can be active at the same time.
    The default value for al-extents is 1237, with a minimum of 7 and a maximum of 65536.
    activity log中的数据以4MB为单位分段存放,al-extents的参数就设定有多少个段可以同时被激活,默认值为1237


    Note that the effective maximum may be smaller, depending on how you created
    the device meta data, see also drbdmeta(8)
    The effective maximum is 919 * (available on-disk activity-log ring-buffer area/4kB -1),
    the default 32kB ring-buffer effects a maximum of 6433 (covers more than 25 GiB of data)
    We recommend to keep this well within the amount your backend storage and
    replication link are able to resync inside of about 5 minutes.
    有效最大值是否过小取决于drdb分区初始化的写入drbdmeta信息
    默认有效的最大值是 919*(ring-buffer area/4KB -1)
    ring-buffer area=--al-stripes * --al-stripe-size-kB
    默认--al-stripes=1,--al-stripe-size-kB=32
    默认有效最大有效值是1*32KB/4KB -1 = 6433,(包含25G的数据, 6433*4MB=25.732GB)
    因为主节点崩溃的情况下activity log需要被恢复,所以推举设置一个值使得backend storage能在5分钟内重新同步完
    If the application using DRBD is write intensive in the sense that it frequently issues small writes scattered across the device, it is usually advisable to use a fairly large activity log. Otherwise, frequent metadata updates may be detrimental to write performance.
    如果你的drbd使用在一个密集写入应用上,那么系统将频繁把小数据写零散的分配到磁盘上
    这种情况下,过多的metadata更新会降低性能.适当的增加activity log大小将能有效的减少metadata更新


    al-updates
    DRBD's activity log transaction writing makes it possible, that after the
    crash of a primary node a partial (bit-map based) resync is sufficient
    to bring the node back to up-to-date. Setting al-updates to no might increase
    normal operation performance but causes DRBD to do a full resync
    when a crashed primary gets reconnected. The default value is yes.
    在drbd activity log的事物写入的支持下,主节点崩溃后只需要重新同步部分数据即可恢复状态
    al-updates设置为no将提高普通操作的性能,但是代价主节点崩溃的情况下需要同步所有数据,默认值yes

    With this parameter, the activity log can be turned off entirely (see the al-extents parameter).
    This will speed up writes because fewer meta-data writes will be necessary,
    but the entire device needs to be resynchronized opon recovery of a failed
    primary node. The default value for al-updates is yes.
    使用这个参数, activity log可以被完全清除,
    这样会提高写入速度, 因为只需要很少的metadata
    但是,如果需要恢复一个失败的主节点节点,这时需要同步整个设备(而不是只更新一些meta-data和具体数据块)
    相当于默认yes是支持事物的innodb, no是不支持事物的myisam


总结一下

        activity log是一个drbd维护的用于记录最近活动数据区域的数据
        activity log可以理解为索引,如果不存在索引,那么重建的时候需要全盘同步
        如果存在这个索引,只要同步索引索包含的数据位置即可
        顺便,因为metadata肯定是在活动区域的,所以同步的时候必须完整同步activity log

        al-updates为no的情况下,相当于关闭activity log.drbd不再维护热点区域
        也就是说除了直接写入数据外只写很少量的meta-data,这时数据写入方式将是非事物式的
        这种情况下drbd节点崩溃后数据不完整,恢复的时候需要同步整个磁盘


        顺便说下,如果文件不在活动区域内,即使是touch也速度也会延迟
        例如网站每天产生很多日志文件,这些日志文件很久不被访问,那么自然不在活动区域(activity log)内
        当删除他们的时候,drbb先要把他们(标记到)activity log区域,然后才能删除
        这个过程会导致rm、touch等的时间变长

---

    on-congestion
    congestion-fill
    congestion-extents
    By default DRBD blocks when the available TCP send queue becomes full. That means it will slow down the application that generates the write requests that cause DRBD to send more data down that TCP connection.
    In an environment where the replication bandwidth is highly variable (as would be typical in WAN replication setups), the replication link may occasionally become congested. In a default configuration, this would cause I/O on the primary node to block, which is sometimes undesirable.
    Instead, you may configure DRBD to suspend the ongoing replication in this case, causing the Primary’s data set to pull ahead of the Secondary. In this mode, DRBD keeps the replication channel open?—?it never switches to disconnected mode?—?but does not actually replicate until sufficient bandwith becomes available again.
    这3参数不用看了,在广域网环境用drbd的时候才要设置的,内网带宽足够不需要管这几个配置,默认是当tcp发送队列满的情况下drbd抢占带宽

    c-plan-ahead plan_time, c-fill-target fill_target, c-delay-target delay_target, c-max-rate max_rate, c-min-rate
    都是和带宽相关的,内网双主的负载量不用考虑


    connect-int 10 # 重连尝试间隔 默认值10 单位秒
    ping-int 8  #心跳ping间隔 默认值10 单位秒
    ping-timeout   #心跳ping超时 默认值500ms 单位毫秒
    timeout 30  #这个参数就是就是其他节点回包超过这个时间就认为节点down,默认值60,单位0.1秒


    wfc-timeout time
    Wait for connection timeout. The init script drbd(8) blocks the boot process until the DRBD resources are connected. When the cluster manager starts later, it does not see a resource with internal split-brain. In case you want to limit the wait time, do it here. Default is 0, which means unlimited. The unit is seconds.
    degr-wfc-timeout time
    Wait for connection timeout, if this node was a degraded cluster. In case a degraded cluster (= cluster with only one node left) is rebooted, this timeout value is used instead of wfc-timeout, because the peer is less likely to show up in time, if it had been dead before. Value 0 means unlimited.
    outdated-wfc-timeout time
    Wait for connection timeout, if the peer was outdated. In case a degraded cluster (= cluster with only one node left) with an outdated peer disk is rebooted, this timeout value is used instead of wfc-timeout, because the peer is not allowed to become primary in the meantime. Value 0 means unlimited.
    不用设置我们不用init.d脚本启动drbd
