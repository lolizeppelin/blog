---
layout: post
title:  "Linux ntp避免时间回滚"
date:   2017-01-16 15:05:00 +0800
categories: "网络基础"
tag: ["linux"]
---

* content
{:toc}

先贴几个参数翻译

    -q      
    Exit the ntpd just after the first time the clock is set. This behavior
    mimics that of the ntpdate pro-gram, which is to be retired.
    The -g and -x options can be used with this option.
    Note: The kernel time discipline is disabled with this option.

    在第一次设置时钟后退出ntpd。 这种行为模仿了(过时)ntpdate程序的行为。
    -g和-x选项可用于此选项。 注意：使用此选项禁用内核时间规则。

解释：
用过时形容ntpdate是因为要尽量避免用ntpdate来同步时间,用ntpdate来同步时间在现代操作系统里是不推荐的
禁用内核时间规则的意思是, 普通的时间设置命令和接口(clock之类)是不能影响内核中的时间计数器的,比如monotonic的返回值
使用ntpd -q就不受到内核时间规则的约束,简单来说就ntpd设置时间影响到monotonic的返回的时间(clock设置时间不影响monotonic),
如果需要获取最绝对的不被影响的内核时间计数器值,需要用monotonic_raw


    -g      
    Normally,  ntpd exits with a message to the system log if the offset
    exceeds the panic threshold, which is 1000 s by default.
    This option allows the time to be set to any value without restriction;  
    however,this  can  happen  only once. If the threshold is exceeded after that,
    ntpd will exit with a message to the system log. This option can be used
    with the -q and -x options. See the tinker  command  for  other options.

    通常，如果时间偏移超过了紧急阈值，那么ntpd退退出并在系统日志中记录，这个紧急阈值默认值为1000秒。
    使用这个选项将允许把时间设置为任何值而不受限制，但是，这种情况只能发生一次。
    如果时间偏移量阈值再被超过一次，ntpd将退出一条消息给系统日志。
    该选项可以与-q和-x选项一起使用。有关其他选项，请参阅tinker命令。

解释：
这个-g用于系统刚启动的时候时间和ntp服务器偏差较大的时候第一次同步时间用
在时间偏差较大的情况下,如果用ntpd不带上这个参数去同步时间,第一次同步就发生偏移过大ntpd进程直接退出
建议在没有应用启动的情况下,先用ntpd -q -g去同步一次时间后,再启动ntpd服务,也可以-g写在ntpd启动配置里/etc/sysconfig/ntp


    -x      
    Normally, the time is slewed if the offset is less than the step threshold,
    which is 128 ms by default,and stepped if above the threshold.
    This option sets the threshold to 600 s, which is well  within  
    the accuracy window to set the clock manually.
    Note: Since the slew rate of typical Unix kernels is limited to 0.5 ms/s,
    each second of adjustment requires an amortization interval of 2000 s.
    Thus, an adjustment as much as 600s  will take almost 14 days to complete.
    This option can be used with the -g and -q options.
    See the tinker command for other options.
    Note: The kernel time discipline is disabled with this option and
    the step threshold is applied also to leap second corrections


    通常情况下, 如果单词偏移量小于默认值为128毫秒的阈值的时候,
    使用slewed(漂移调整,微调)调整时间,如果超过128毫秒,将使用stepped(跳跃调整,大调)方式调整时间
    这个参数将会将阈值设置为600s(这个阈值的精准窗口适用于常规的时间设置)
    也就是说,当偏移量大于600秒的时候,才会使用stepped方式调整时间,600秒以内都使用微调的方式
    注意：典型的unix内核将时间摆动量限制在0.5ms/s(也就是微调每秒最多能偏移0.5ms)
    也就是说,使用-x参数后,时钟要调整一秒钟最少需要累计2000秒才能凑够（2000*0.5）
    所以,如果偏移量是600s, 那么需要14天才能完成时间同步.
    此选项可与-g和-q选项一起使用。有关其他选项，请参阅tinker命令。
    注意：使用此选项禁用内核时间规则，单次阈值也适用于闰秒的修正

解释：
默认微调的偏移量是128ms以下,超过128ms使用跳跃时间调整
启用了这个参数以后,偏移量在600s以内都是微调,600s以上才跳跃调整
启用这个选项可以让系统时间在600s以内都不会跳跃,也就是说不会被回退
600s-1000s之间还是会出现时间跳跃,有可能出现时间backwards

由于unix内核中限顶了微调的上线时每秒最多偏移0.5ms,启用这个参数后如果时间变动较大但是有在600s内
那么需要很长的时间,时间才能完成同步,这部分[参考](http://www.happyworld.net.cn/post/6.html)

---

建议启用-x这个选项避免时间跳跃

由于启动了这个参数以后,偏移在还会在600-1000的时候时间还是会stepped跳跃调整(有可能回滚)
所以我们要将1000这个让ntp挂掉的阈值调低到600

    #设置
    tinker panic 600

还有一种办法防止时间stepped调整的时候回滚。[参考](https://stackoverflow.com/questions/35068445/is-there-a-way-to-ensure-ntp-synced-clock-never-moves-backwards)

    #设置
    tinker stepback 0

由于需要高版本的linux/ntp才能支持,适用性不强
