---
layout: post
title: "Hadoop之Yarn的内部机制"
subtitle: "Yarn内部机制和HA"
date: 2021-01-11
author: "Hanke"
header-style: "text"
tags: [Hadoop, Yarn, BigData]
---
> 前面两篇文章，我们介绍了Hadoop里两个重要的组件[MapReduce](/2021/01/04/MapReduce-in-Hadoop/)和[HDFS](/2021/01/11/hdfs-in-hadoop)。本文我们一起看一下，作为大数据业内用的比较普遍的Yarn的内部机制。

## Hadoop 1.x 和 2.x的设计对比
首先让我们从总体上看一下Hadoop 1.x和2.x设计的不同之处。
![Hadoop V1 and V2](/img/hadoop/yarn/HadoopV1_V2_arch.png)

从Hadoop V1和V2的总体设计对比上，可以明显看到几个变化:
* 增加了Yarn组件。
* 1.x的Map Reduce将资源的管理调度和数据的处理计算进行了拆解，仅保留了数据处理计算的功能，而用Yarn来进行集群的资源管理和调度工作。
* 平台能够支持除了MapReduce以外的其他计算框架，比如Spark, Flink等的计算任务。 

## 为什么会出现Yarn?
那么是什么样的原因在Hadoop 2.x里引入Yarn这个如今这么受欢迎的组件呢？接下来，我们从更细的架构及工作机制来了解一下背后的故事。

### 旧版MapReduceV1架构
在MapReduceV1，采用的是经典的Master-Slave架构，可参考下图。
![Hadoop V1架构](/img/hadoop/yarn/HadoopV1_arch.png)

在架构图中存储和计算部署在一个集群里，其中Master机器上有JobTracker和NameNode，而Slave机器上有TaskTracker和DataNode。其中关于存储部分NameNode和DataNode在[HDFS文章](/2021/01/11/hdfs-in-hadoop)上有介绍，这里就不再赘述。

MapReduceV1里计算部分由两个重要组件`JobTracker`和`TaskTracker`组件:
* **JobTracker** 职责
    * 集群里大脑的角色。
    * 负责集群的资源的管理和数据处理任务的调度。
    * 负责任务分配与管理: 任务的分发、状态监控、重启等。

* **TaskTracker** 职责
    * 协助JobTracker的工作，具体数据处理任务执行的实体。
    * 从JobTracker处领取要执行的任务。
    * 向JobTracker发送heartbeat汇报机器的资源状态和任务执行情况。

### 旧版MapReduceV1存在什么样的问题？
从上面V1架构图来看，整体的Master-Slave工作模式还是比较简单清晰，Hadoop在发布之初也得到业内的快速认可。但随着时间的推移，越来越多的问题凸显出来。  

用一个大家熟悉的场景来比喻的话，V1里的JobTracker类似于一个公司的老板，而TaskTracker就好比公司里具体干活的每一个员工。在V1里，老板需要事无巨细的了解人力资源使用情况，为公司接的每个项目分配人力、需要做所有项目的管理工作、并且需要了解每个项目里每个参与的员工具体的工作执行情况: 进度，是否需要换人等等。另外，公司员工可以做的工作的工种也比较单一，只能做Map或者Reduce两类工作。  

当公司规模比较小的时候，比如刚刚创业时只有几十人时，老板作为这个公司的大脑层，跟的这么细还能承受住。但是当公司越做越大时，比如膨胀到几千人以后，如果老板还需要每天都了解具体每一项小工作的执行情况，肯定会累的吐血....这也是当时总结出来的V1最多只能支持4000节点的集群规模上限。  

总结一下MapReduceV1存在的**问题**:
* JobTracker是单点瓶颈及故障，负担太重。
* 只支持Map、Reduce两类计算task类型，和Hadoop本身的框架耦合比较重，无法支持其他的计算类型，复用性较差。
* 资源划分是按Map Slot、Reduce Slot进行划分，没有考虑Memory和Cpu的使用情况，资源利用有问题（容易OOM或者资源浪费）。

### 新组件Yarn的架构
为了解决上面的问题，Hadoop在2.x里重新做了设计，引入新的组件Yarn。其中最重要的变化是把V1里JobTracker职责**资源管理调度和数据计算任务管理**两部分进行了拆分，Yarn主要用来负责资源管理调度，与具体的计算任务无关。而[2.x](#hadoop-1x-和-2x的设计对比)里的MapReduce只负责Hadoop相关的数据处理和计算等相关工作。Yarn组件的出现，极大提升了Hadoop组件的复用性，能够很容易支持其他计算框架的任务。

我们具体看一下Yarn的架构:



## Yarn的HA

## Reference
* [Hadoop文档][1]

<b><font color="red">本网站的文章除非特别声明，全部都是原创。
原创文章版权归数据元素</font>(</b>[DataElement](https://www.dataelement.top)<b><font color="red">)所有，未经许可不得转载!</font></b>  
**了解更多大数据相关分享，可关注微信公众号"`数据元素`"**
![数据元素微信公众号](/img/dataelement.gif)

[1]: https://hadoop.apache.org/docs/current/
