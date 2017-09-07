---
layout: post
title:  "taskflow 使用指南-1"
date:   2017-09-06 15:05:00 +0800
categories: "编程"
tag: ["openstack", "python"]
---

* content
{:toc}


taskflow是openstack未确保冗长的系列操作能正确执行的工作流框架

    主要应用场景有如 Cinder 的 create volume 这般复杂、冗长、容易失败, 却又要求保持数据与环境一致的业务逻辑.

由于适用场景非常多,例如游戏行业的常见操作

    停服 备份数据库 升级数据库  升级应用 启动服务器

非常适合用工作流引擎来确保执行

这个由openstack编写的taskflow国内相关介绍非常少,官方用户指南内容太多,导致学习这个框架花费需要花费非常多的时间

### 这里将会较为详细的介绍taskflow的代码和使用


百度有一篇中文的[相关介绍](http://blog.csdn.net/jmilk/article/details/60496748),作者对taskflow还是很熟的,但是写得太简要,直接看感觉明白了实际还是一头雾水,可以预读一下

要看懂taskflow,需要先熟悉两个最基本的概念


1. 工作流  这玩意最直观的一个应用就是oa里审批,上面我们说的游戏业的常见操作也可以作为例子

    停服 备份数据库 升级数据库  升级应用 启动服务器

    上述操作也是一个工作流, 停服成功才能备份,备份成功才能升级，
    升级应用和备份可以同时进行,升级数据库和升级应用都成功了才能启动服务器.
    升级失败还要自动回滚.

2. 有限状态机  推举[导读](http://www.jianshu.com/p/5eb45c64f3e3).
   工作流就是用状态机来实现流程控制和执行任务的


 上面两条光看懂了还不行,还要写过一轮深入了解,taskflow使用了openstack的通用状态机项目,所以要能看taskflow先要熟悉openstack的通用状态机项目[Automaton](https://pypi.python.org/pypi/automaton/)


为了熟悉Automaton,我用它写了一个[tailfile](https://github.com/lolizeppelin/tailfile),作用就是实现tail -f的效果,倒读N行和判断文件是否变化都用到状态机,因为状态和事件都比较少,用Automaton写这些功能有点杀鸡用牛刀,写这个项目的主要目的是用来熟悉状态机.

熟悉完Automaton以后,可以开始来看taskflow(基于最新版本2.14)了。

首先,因为taskflow作为通用项目而不是openstack内部项目,所以taskflow没有使用oslo中的一些通用工具

而且里面同样为了兼容写了一些复杂的接口还用了futurist实现异步,我为了和自己的其他几个项目统一起来,基于taskflow2.14生成了一个简化过的项目[simpleflow](https://github.com/lolizeppelin/simpleflow)



代码变化了几个如下部分

1. 日志用回oslo_log的方式
2. 删除了sqlalchemy以外的存储方式,table的和所有sql expression都改为orm方式（只支持mysql）,后来我熟悉taskflow以后发现删除其他backends只保留mysql不是一个正确做法,有缺陷,后续会说明为什么
3. 因为删除了sqlalchemy以外的存储方式, 所以persitence中backends就么有存在的必要了,直接传sqlalchemy的session即可
4. persitence中的内容和storage合并
5. jobs和conductors删除,这个部分可以和主要的taskflow功能无关
6. Executor不使用futurist,自己写了一个简化的futurist,代码兼容原来的写法,只支持协程,删除多线程和多进程相关支持
7. 读写锁业复制了过来,用协程实现
8. 砍掉了worker based的Engine

因为极大的简化了futurist和删掉了backends层所以比较好读一些,读代码的时候建议看我简化后的项目


我们先来看taskflow的几个主要单位

1. Engine

    Engine是taskflow是启动口, 主要工作 1.创建状态机, 2.循环状态机, 3.在指定状态下通过Executor执行任务, 4.其他接口（存储、调度器等）的初始化

    Engine分为好几种. work based的Engine比较特殊我们不看,只看action engine

    其实几种action engine其实没有什么区别,通过Executor分为

    1. 序列化（阻塞,顺序执行）引擎
    2. 基于线程的并行引擎
    3. 基于协程的并行引擎
    4. 基于多进程的并行引擎

    并行引擎的优势是,当个任务没有顺序关联的情况下可以同时执行多个任务

    当然,引擎不影响任务之间的顺序关系,除非你想强制一个一个任务执行,否则都应该使用并行引擎

    我自己的代码里简化以后只剩下序列化引擎和协程引擎,Executor是什么看下面


2. Executor

    前面说了,Engine会通过Executor执行任务,因为如果Engine直接执行任务的话,整个状态机的循环会受到正在执行的任务的影响,所以包了一层Executor来执行具体的任务(当然具体代码里对Executor的应用会更复杂一点,为了扩展和异常处理包了3层)

    在taskflow的源代码中Executor是通过futurist库来实现的,而futurist又是基于futures的,这个库内部实现还是比较复杂的,如果没用过对应库的,建议直接参考我简化的[futurist](https://github.com/lolizeppelin/simpleutil/blob/master/simpleutil/utils/futurist.py),因为是用eventlet实现的,所以需要熟悉eventlet.

    具体的任务代码（比如备份数据库什么的）在一般情况下可以不处理异常,因为执行任务的代码通过except Exception捕获了任务的所有异常.

    特殊异常就是CancelledError,这个异常是调度到已经取消任务时由futurist抛出,在读代码的时候需注意下这个的特殊处理

3. Scheduler

    这个没什么好说的,Executor的封装的最上层,最后执行会落实到具体的Executor上

4. storage

    这个是存储接口,后面说到flow的时候会详细讲到,storage的初始化在Engine中,一个功能是数据存储的接口,一个功能是作为flow的外层封装

4. Runtime与machine

    在看这个之前,如果你还不熟悉状态机,建议先拿前面说的automaton练练手,如果已经熟悉状态机但是还没看过automaton代码的，建议去看看automaton的代码

    machine就是Engine中循环的(automaton)状态机了,一个engine只运行一个状态机,初始化代码在builder.MachineBuilder,MachineBuilder又是在Runtime中调用生成machine的,我们先别管Runtime,先理解一下taskflow的状态机

    taskflow状态机并不复杂,但还不熟悉taskflow的时候很容易被高懵.因为taskflow用到networkx这个图库,而状态机其实就是一个有向图,所以一开始看的时候,很容易以为taskflow的状态机会非常复杂要看懂图的相关代码才能搞明白,但实际情况是

    taskflow的状态机和图无关!因为taskflow状态机的状态很少不需要用图来解决状态循环

    那么taskflow为什么要用到图库呢,在解决这个疑问前我们先抛开taskflow,自己用状态机设计一个解决前面——"停服 备份数据库 升级数据库  升级应用 启动服务器" 的工作流

    1. 首先定义停服状态和对应停服状态执行的代码
    2. 定义停服成功的返回,失败的返回,定义进入停服状态的event（这个是起始时间,event就是start）
    3. 定义备份数据库状态对应备份执行代码
    4. 定义进入备份状态的event（前面的停服成功）
    5. 定义备份成功和备份失败的返回,到目前还简单,备份失败大不了多备份几次直到成功,失败了整个状态机终止都可以影响不大
    6. 定义升级数据库状态对应备份执行代码
    7. 定义进入升级状态的event（前面的备份成功）
    8. 定义升级成功和备份失败的返回,这里开始坑了,升级失败要回滚了
    9. 发现少了回滚升级失败的状态定义.....增加升级失败回滚失败的定义
    ......
    回滚升级数据库失败....升级应用是失败...回滚升级应用是失败.....启动失败

    设计下来你发现没几个步骤。要定义的状态就越来越多...这也就是状态机复杂以后和图有关的原因了

    taskflow非常巧妙的避免了复杂化状态机,taskflow的设计的状态机可以简单的理解只处理2个状态就好

    开始....找到任务-执行任务-找到任务....执行任务...终止

    执行任务就是调用Executor, 至于找下一个任务的工作,就是封装了图库的flow的工作了.这样设计状态机状态就很少,具体的状态可以看MachineBuilder的注释中有对应表格,对应状态目前粗看一下即可,知道哪个状态是找任务、哪个状态执行任务就可,有些状态涉要看了后面的retry相关才比较好理解,至于flow,这个我们在后面说明

    回头来看Runtime,MachineBuilder是由Runtime生成的,状态机的有些callback最终执行的Runtime中的函数,里面会有一些嵌套和封装, Scheduler的封装就在Runtime中,Runtime可以简单理解为状态机调用其它诸如Scheduler,storage等接口的中间件,Runtime在整体理解taskflow的的时候可以不用细看

第一篇完...请看下一篇介绍flow  atom task retry
