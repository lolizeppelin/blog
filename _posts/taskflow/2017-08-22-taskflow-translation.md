---
layout: post
title:  "taskflow用户指南翻译-1"
date:   2017-08-22 15:05:00 +0800
categories: "编程"
tag: ["openstack", "python"]
---

* content
{:toc}


1.Atom

    atom是TaskFlow中最小的单元,他是其他所有类的基类,atom有一个name的属性,
    还有一个版本号作为可选属性
    atom的name通过输入指定,name值可以输出


```python
class taskflow.atom.Atom(name=None, provides=None, requires=None, auto_extract=True, rebind=None, inject=None, ignore_list=None, revert_rebind=None, revert_requires=None)[source]
    """
    基类: object

    An unit of work that causes a flow to progress (in some manner).
    一个让flow以某种方式进行的工作单位

    An atom is a named object that operates with input data to perform some action that furthers the overall flows progress. It usually also produces some of its own named output as a result of this process.

    atom是被命名过,通过输入的数据来控制的对象
    atom用于执行一些动作来确保整体flow正确运行
    它通常产生一些由自身命名的过程结果输出

    Parameters:

    name – Meaningful name for this atom, should be something that is distinguishable and understandable for notification, debugging, storing and any other similar purposes.
    一个有意义的名字,必须在通知、调试、存储等行为中可辨识和可以理解的

    provides – A set, string or list of items that this will be providing (or could provide) to others, used to correlate and associate the thing/s this atom produces, if it produces anything at all.
    可以是set、string或者items组成的列表, 这些对象将提供或者可以提供给其他（对象）
    用于将当前atome和这些对象关联,如果这些对象能产生任何东西的话

    inject – An immutable input_name => value dictionary which specifies any initial inputs that should be automatically injected into the atoms scope before the atom execution commences (this allows for providing atom local values that do not need to be provided by other atoms/dependents).
    一个不可变的name->value字典, 他指定了必须在atome执行之前自动注入到当前atom区域的初始化输入值
    通过这种方式允许提供给atom一个不需要提供给其他atom/dependents的local本地值

    rebind – A dict of key/value pairs used to define argument name conversions for inputs to this atom’s execute method.
    一个k/v字典, 用于提供参数转换atome的execute method参数名

    revert_rebind – The same as rebind but for the revert method. If unpassed, rebind will be used instead.
    一个k/v字典,提供参数给revert method用,revert method未能通过,rebind的的值将用于revert method
    1.3版本里没有这个参数

    requires – A set or list of required inputs for this atom’s execute method.
    一个set/list, 用于指定execute method运行必要的输入

    revert_requires – A set or list of required inputs for this atom’s revert method. If unpassed, `requires will be used.
    同上,用于revert method, 为你能通过使用requires的值
    1.3版本里没有这个参数


    Variables:
    version – An immutable version that associates version information with this atom. It can be useful in resuming older versions of atoms. Standard major, minor versioning concepts should apply.
    atom的版本号,不可变,在恢复旧版atomes的时候会非常有用.主版本和次要版本的概念是适用的
    也就是允许 a.b 的版本号形式,a表主要版本,b表次要版本


    save_as – An immutable output resource name OrderedDict this atom produces that other atoms may depend on this atom providing. The format is output index (or key when a dictionary is returned from the execute method) to stored argument name.
    一个不可变的输出资源名字典, 这个atom提供的资源是其他的atome可能依赖的
    格式是输出索引（如果execute method返回的是字典,那么输出的就是字典的key）到存储参数名
    1.3版代码中save_as是一个有序字典（OrderedDict）


    priority = 0
    A numeric priority that instances of this class will have when running,
    used when there are multiple parallel candidates to execute and/or revert.
    During this situation the candidate list will be stably sorted based on
    this priority attribute which will result in atoms with higher priorities
    executing (or reverting) before atoms with lower priorities (higher being
    defined as a number bigger, or greater than an atom with a lower priority number).
    By default all atoms have the same priority (zero).
    For example when the following is combined into a graph (where each node in the denoted graph is some task):
        a -> b
        b -> c
        b -> e
        b -> f
    When b finishes there will then be three candidates that can run (c, e, f)
    and they may run in any order. What this priority does is sort those three
    by their priority before submitting them to be worked on (so that instead
    of say a random run order they will now be ran by there sorted order).
    This is also true when reverting (in that the sort order of the potential
    nodes will be used to determine the submission order).
    这就不详细翻译了,简单说明下
    a执行完后执行b, b执行完后可以执行c e f,执行顺序是不确定的,
    设置了priority, c e f的执行顺序就有优先级了
    还原的时候优先级也有效


    requires – A OrderedSet of inputs this atom requires to function.
    通过上面参数requires生成的有序字典 key就是参数名字, value是参数别名
    默认情况下key==value, key!=value的情况看下面rebind

    rebind – An immutable input resource OrderedDict that can be used to
    alter the inputs given to this atom. It is typically used for mapping
    a prior atoms output into the names that this atom expects
    (in a way this is like remapping a namespace of another atom into the namespace of this atom).
    这里不翻译了,看下面的代码说明

    revert_rebind – The same as rebind but for the revert method. This should only differ from rebind if the revert method has a different signature from execute or a different revert_rebind value was received.
    inject – See parameter inject.
    Atom.name – See parameter name.
    requires – A OrderedSet of inputs this atom requires to function.
    optional – A OrderedSet of inputs that are optional for this atom to execute.
    revert_optional – The revert version of optional.
    provides – A OrderedSet of outputs this atom produces.
    相当于返回值的作为关联task的参数名,和上面rebind相关
    """

```

---

看完代码总结下

    通过反射executor（一般就是继承类的self.execute方法和revert方法）
    获取到executor没有默认值的参数和有默认值的参数
    然后自动定义有默认值的为可选参数(属性optional),无默认值为必要参数(属性requires)
    因为有默认值的参数在外部可能也是必要参数,所以通过初始化传入的requires可以指定必要参数
    通过计算后,上面的必要参数字典(requires)中增加,可选参数字典(optional)减少
    rebind就是把参数名映射成其他名字,比如executor(a, b)
    在默认情况下requires字典值是{'a': 'a', 'b': 'b'}, 通过rebind=['a1', 'b1']
    requires字典将变为{'a': 'a1', 'b': 'b1'}
    这样关联任务设置provides为a1的时候,返回后这边task就能获得参数a

    save_as  这个有序字典是通过迭代前面的参数provides生成
    provides 有序set,从save_as中取出的所有key生成,也相当于一开始的参数provides存到有序set中
    具体看_save_as_to_mapping就知道了

    provides
    这个属性是任务链的关键参数,但是这里理解不了,需要后面才能看懂,先看看翻译就好
---


Engines
---

    Overview
    Engines are what really runs your atoms.

    An engine takes a flow structure (described by patterns) and uses it to decide which atom to run and when.

    TaskFlow provides different implementations of engines. Some may be easier to use
    (ie, require no additional infrastructure setup) and understand; others might
    require more complicated setup but provide better scalability.
    The idea and ideal is that deployers or developers of a service that use
    TaskFlow can select an engine that suites their setup best without modifying the code of said service.

    真正执行atomes的就是Engines

    Engines需要一个由patterns决定流式的结构,通过这个结构决定哪个atom在什么时候被执行
    注：这里的patterns就是代码中的patterns文件夹中的有向图flow、无向图flow
        对图库networkx使用就是在这里了

    TaskFlow提供了集中不同的engines,有些易用性好,有些需要复杂的设置,但是扩展性好
    这里的设计理念是：或开发人员可以选择最适合其设置的引擎，而无需修改所述服务的代码。


    Engines usually have different capabilities and configuration,
    but all of them must implement the same interface and preserve
    the semantics of patterns (e.g. parts of a linear_flow.Flow are run one after
    another, in order, even if the selected engine is capable of running tasks in parallel).

    引擎通常具有不同的功能和配置，但是所有这些都必须实现相同的接口并保留模式的语义
    （例如，一个linear_flow的部分.Flow依次运行，即使所选择的引擎能够并发执行任务）。
    这里就是说,无论用什么Engine, 主要参数是一直的


    An engine being the core component which actually makes your flows progress
    is likely a new concept for many programmers so let’s describe how it operates
    in more depth and some of the reasoning behind why it exists.
    This will hopefully make it more clear on their value add to the TaskFlow library user.

    engine作为核心部件对使用面向过程编程的程序原来说是一个新概念
    所以这里需要更深入的描述它是如何工作的,然后推到出它存在的意义
    这将有助于使用taskflow的用户了解engine的价值


Task

    A task (derived from an atom) is a unit of work that can have an
    execute & rollback sequence associated with it (they are nearly analogous to functions).
    Your task objects should all derive from Task which defines
    what a task must provide in terms of properties and methods.

    Task（继承自atom）是一个工作单位,一个可以执行和回滚的序列可以关联到它,行为类似函数
    所有自定义的task对象应该继承自Task
    Task类定义了必要的属性和方法



Retry

    A retry (derived from an atom) is a special unit of work that handles errors,
    controls flow execution and can (for example) retry other atoms with other parameters if needed.
    When an associated atom fails, these retry units are consulted to determine
    what the resolution strategy should be. The goal is that with this consultation
    the retry atom will suggest a strategy for getting around the failure
    (perhaps by retrying, reverting a single atom, or reverting everything contained in the retries associated scope).

    Currently derivatives of the retry base class must provide a on_failure() method
    to determine how a failure should be handled. The current enumeration(s)
    that can be returned from the on_failure() method are defined in an enumeration class described here:

    ---


翻译已经坑了,翻译是为了能用taskflow,但是不如直接看代码好用....所以请看[taskflow使用指南-1](http://www.lolizeppelin.com/2017/09/06/taskflow-howto/)
