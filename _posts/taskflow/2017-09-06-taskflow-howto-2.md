---
layout: post
title:  "taskflow 使用指南-2"
date:   2017-09-06 15:05:00 +0800
categories: "编程"
tag: ["openstack", "python"]
---

* content
{:toc}


这篇指南主要介绍taskflow中的atom与task


## atom与task

> 在atom的用户指南中,ATOM的说明是——atom是TaskFlow中最小的单元XXXXXXX
>> 这句话比较唬人,实际上只有两个单位继承自atom,一个是Task，一个是Retry.状态机里的任务必须继承自Task,至于Retry我们放到后面在说这个它的作用,而task是atom的简单封装,所以我们直接先讲task
> 在task初始化里,有几个参数非常重要需要侧重理解：
>> 1:name, 这个不用解释,不管你是task还是retry,必须有一个名字,一般用于识别你这个任务的作用,比如dump-db什么之类的,如果不设置会自动用类名来生成，由于一个flow中不能添加同名的atom,所以有时候必须设置对应的name
>> 2:rebind, 这是一个非常关键的参数,需要用一段代码来解释

```python
from simpleflow import task

class MysqlDump(task.Task):
    def execute(self, a, b, c=None)：
        """do dump database here"""
        pass

    def revert(self, result, *args, **kwargs):
        """do revert here"""
atask = MysqlDump()
```

>>>当我们需要在taskflow中执行一个任务的时候, 常规的方式是,继承task.Task
然后重写execute方法,execute中的代码就是我们需要具体执行的任务
当我们的execute需要接收参数的时候,参数名就是a和b。

>>>现在我们来看看Task的基类atom中的_init__,
它调用的atom.py最上面的_build_arg_mapping和_build_rebind_dict是非常关键的函数
这里就不贴代码了,我直接说最终结果

>>初始化的时候,通过反射execute函数获取到execute的参数列表及参数对应的默认值
有默认值的为可选参数,没有默认值的是必要参数,存入一个有序字典中

>>>flow通过记录这个有序字典,当有对应参数出现（由前一个任务生成）的时候,就会调用这个task

>>>也就是说,对于在flow中的atask（上面没有写添加进flow的代码）
如果发现有a, b参数生成（c是必要参数）,那么atask会被调用
（这里只是简单描述下flwo的调用,实际调用会比较复杂）

>>>这就有一个问题,如果有多个MysqlDump实例,都用只能用 a,b参数来"激活"
那也就太不灵活和合理了,于是有了这么一个参数rebind用于生成的参数名映射
而flow也能通过映射的参数来调用task

```python
atask = MysqlDump(rebind=["a1", "a2"])
btask = MysqlDump(rebind=["b1", "b2", "b3"])
```

>>>上述代码就能实现出现a1, a2参数的时候atask
被调用,出现b1,b2,b3参数的时候,btask被调用(rebind会把可选参数变为必要参数)


>> 3:provides 英文的原意是提供的意思,前面我们说rebind的时候有提过上前一个任务提供参数,provides就是表示当前任务的execute的执行结果能提供什么参数.

>>默认情况下provides为None,也就是说无论execute的执行结果是什么,当前task都不提供任何参数到外部

```python

class MysqlDump(task.Task):
    def execute(self, a, b, c=None)：
        return a+1, b+1, c+1 if c else 0

atask = MysqlDump(provides=["b1", "b2", "b3"], rebind=["a1", "a2"])
btask = MysqlDump(rebind=["b1", "b2", "b3"])
```

>>> 上面的代码表示,atask能输出参数b1, b2, b3
当然,provides中内容必须少于或者等于execute返回的内容
我们可以通过provides和rebind来让任务之间产生关联并按顺序执行,provides和rebind的应用在taskflow的例子中有一个很好的体现,用例[参考](https://github.com/lolizeppelin/simpleflow/blob/master/doc/examples/graph_flow.py)
>>>这个例子适合反复参考来理解provides和rebind,不过看这个例子之前你还需要了解一个特殊的task, 名字叫_TaskFlow_INJECTOR
>>>task的参数可以是上一个任务通过provides提供的,如果有初始参数的提供,taskflow通过一个名为_TaskFlow_INJECTORD的task来provides参数,当有store(例子中run传入的store)参数的时候, _TaskFlow_INJECTOR是所有task的前置task.

请看下一篇介绍flow
