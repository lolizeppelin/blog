---
layout: post
title:  "taskflow 使用指南-3"
date:   2017-09-07 15:05:00 +0800
categories: "编程"
tag: ["openstack", "python"]
---

* content
{:toc}


这篇指南主要介绍taskflow中的flow和storage


## flow和storage

Engine的必要初始化参数一个是flow,一个就是backends,backends对应的是storage使用何种后端存储,我简化后的代码storage只支持sqlalchemy,所以不直接传入sqlalchemy的session即可,另外一个参数就是我们马上要说的flow

# flow

通过前面两篇我们知道,Engine是通过flow来找到需要执行的task的,所以,我们的task都是add到flow的,例如：

```python
from simpleflow.patterns import graph_flow as gf

class Adder(task.Task):

    def execute(self, x, y):
        print 'do!!!', x, y
        return x + y

gflow = gf.Flow('root')
gflow.add(Adder(name='atask'))
gflow2 = gf.Flow('leaf')
gflow.add(gflow2)
```

当然flow的add参数可以是task,还可以flow.

flow的代码比较复杂,关键属性_graph就是networkx的图了,不建议现在读代码,粗读一部分,稍微看一下对应的图和关键变量是什么就好和add部分就好。

flow一共有三种,使用到有向图和无向图,当flow调用add添加的task/flow的时候,就是在图中增加节点,对于有向图,添加节点后自动增加边,我来详细看看3中flow的具体区别

1. linear flow, 下面简称lf
>使用有向图,图的边用有序字典存储,也就是说,边是有序的  
因为是顺序执行的flow,所以lf其实就是只有一条线的图,其add代码也比较简单   
但是并不是说lf是顺序执行的所以lf就用序列化的引擎就好,因为add的对象可以是work也可以是flow,lf的节点可能是一个可以并发的gf或者uf

2. graph flow, 下面简称gf
>使用有向图,图的边用字典存储,也就是说,边是无序的  
和lf不同,gf才能算真正的图,所以其add代码复杂很多,新add的节点会回溯前面节点的provides,如果你忘记provides是什么了回头去看第二篇  
gf是并行的,但如果后面的节点参数由前面节点的provides,那么这部分还是有序的  

3. unordered flow, 下面简称uf
>无向图,没有边,十分适合并行,比如说stop成功后批量备份数据库


# storage

这里我就直接用我已经砍掉backends层的代码来说明了,我们来看storage里用到的几个对象（taskflow在persistence包,也就持久化包里,我的代码都集中到storage包里了）

1. Connection
    taskflow中,Connection上面还套了一层backends,backends就一个函数get_connection用于返回Connection实例, 因为原来代码可以支持文件、内存等接口,所以有这么一层Connection用于存放通用方法,具体有什么方法可以直接看[代码](https://github.com/lolizeppelin/simpleflow/blob/master/simpleflow/storage/impl.py)我这里就不描述了,方法的名字很明确的描述了方法的作用

2. models中LogBook,FlowDetail,AtomDetail
    原taskflow的models我改为了middleware,这里的models对应的是tables  
    这样改是为了统一orm框架写法,orm框架中都在models里放表结构  
    所以这里的这3个对象对应了三张表  
    LogBook：一次工作流有且只有一个LogBook,其实条记录用处不大,主要用来识别一次工作流,由于外键关系,删除一条logbook将会删除对应的其他表的记录,工作流正常结束后是可以删除相关记录的  
    FlowDetail：一次工作流只能接收一个flow,虽然前面我们flow中可以添加flow,但是并不会生成FlowDetail记录,顶层flow直接会获取下层flow的具体atom  
    AtomDetail: task和retry都是继承自atom,所以AtomDetail的的每一行就对应一个task或retry(retry后面单独说),还有之前说的用于传输初始参数的_TaskFlow_INJECTORD

3. middleware
    middleware里的LogBook,FlowDetail,AtomDetail就是models里表的dict版增加了一些方法,TaskDetail/RetryDetail自然也是继承了AtomDetail,它们都是存到AtomDetail里,为什么要拆两份而不统一起来呢？一个原因是原taskflow了因为要支持各种backends,所以分了一层出来。还有一个原因是在没有写入（task初始化,返回,取消,报错）的的情况下,图的相关动作都在内存中进行,直接用orm的对象太过笨重,middleware中的对应对象就轻量很多

现在我们来看Storage类了

我们看看engine里对Storage的初始化

```python
@misc.cachedproperty
def storage(self):
    def _scope_fetcher(atom_name):
        if self._compiled:
            return self._runtime.fetch_scopes_for(atom_name)
        else:
            return None
    return storage.Storage(self._flow_detail,
                           connection=self.connection,
                           scope_fetcher=_scope_fetcher)
```
connection在上面已经说明了,scope_fetcher先不管,这个_flow_detail我们看看是啥

当我们调用api.run的时,engine室友load函数创建的
```python
def load(connection,
         flow, flow_detail=None, book=None, store=None,
         engine_cls=SerialActionEngine, **options):

    if flow_detail is None:
        flow_detail = p_utils.create_flow_detail(flow, book=book,
                                                 connection=connection)
    try:
        engine = engine_cls(flow, flow_detail, connection, options)
    except Exception:
        raise exc.NotFound("Could not find engine '%s'" % str(engine_cls))
    else:
        if store:
            engine.storage.inject(store)
        return engine
```

可以看到,Storage用的是flow_detail,我们传入engine的是flow,flow_detail为none的情况下通过p_utils.create_flow_detail生成

>flow_detail就是Flow在middleware中的FlowDetail映射    
同理TaskDetail/RetryDetail自然就是Task和Retry的映射

>由于可以同时传入flow和flow_detail,要是我主动生成的flow_detail和flow不一致怎么办...    
不好意思我还没试验过  
flow_detail可以传入估计是为了能自定义logbook,所以看到create_flow_detail可以是可以接收book参数的,还记得前面我们所说的logbook的作用么？


## 现在我们来总结下

>初始化的flow会被转化为一个middleware.FlowDetail实例  
flow中的所有task/retry(包括下层flow中的task/retry)会被转换为middleware.TaskDetail/RetryDetail实例

>唯一个middleware.FlowDetail实例会有一条对应表models.FlowDetail的记录(不是实例只是数据库里的一行)

>多个middleware.TaskDetail/RetryDetail实例会对应到表models.AtomDetail的多条记录,每个middleware.TaskDetail/RetryDetail实例都会关联到middleware.FlowDetail实例

>当需要查找atom的时候,Storage会通过middleware.FlowDetail实例查找到对应的middleware.TaskDetail/RetryDetail实例  
有写入操作的情况下会把middleware.TaskDetail/RetryDetail实例转化为models.TaskDetail/RetryDetail通过orm写入数据库并更新middleware.TaskDetail/RetryDetail实例

>找到对应的middleware.TaskDetail/RetryDetail实例后,通过_browse_atoms_for_execute转为Task和Retry,这里面也比较复杂涉及到execution_graph的生成,这个将在下一篇介绍

---

几个重要的类说明完毕,现在可以认真看看第二篇所说的[例子](https://github.com/lolizeppelin/simpleflow/blob/master/doc/examples/graph_flow.py)了

熟悉例子以后,我们看下一篇,taskflow状态机如何找到下一个task
