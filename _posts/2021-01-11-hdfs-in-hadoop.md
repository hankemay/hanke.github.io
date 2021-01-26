---
layout: post
title: "Hadoop之HDFS内部机制"
subtitle: "HDFS内部机制和HA方案"
date: 2021-01-11
author: "Hanke"
header-style: "text"
tags: [Hadoop, HDFS, BigData]
---
在前一篇"Hadoop的MapReduce到底有什么问题"里，我们一起回顾了MapReduce内部机制和存在的问题。在本文中，主要讨论Hadoop里另外一个重要组件HDFS的架构和高可用相关机制。感兴趣的同学也可进一步阅读官方[HDFS设计文档][1]。

HDFS设计的目的就是分布式环境下海量数据的存储。其中最重要的目标就是：
* 系统的高可用
* 数据一致性
* 高并发  

## HDFS的架构与工作机制
HDFS的架构图如下:   
![HDFS-Arch](/img/hadoop/HDFS-Arch.png)
HDFS主要由Namenode和DataNodes组成:
* **NameNode职责**:
    * 扮演的是整个分布式存储的大脑角色。
    * 存储HDFS所有的metadata信息，比如Namespace的名字，文件的replicas的个数等。
    * 执行所有文件操作系统等的动作并向DataNode发相应的Block指令，比如打开、关闭、重命名、复制等操作。
    * 负责Block和DataNode之间的mapping关系。
> NameNode的角色类似文件系统里的`文件控制块`角色，在linux里文件控制块会记录着文件权限、所有者、修改时间和文件大小等文件属性信息，以及文件数据块硬盘地址索引。 HDFS的Block Size从2.7.3版本开始默认值从64M更改为128M。

* **DataNodes职责**:
    * 响应Client的读写请求。
    * 执行来自NameNode的block操作请求，比如复制，删除，新建等命令。
    * 负责向NameNode汇报自己的Heartbeat和BlockReport。


## HDFS的HA
* **元数据方面**
    * NameNode在HDFS里的重要性不言而喻，如果NameNode挂了或者元数据丢失了，那么整个HDFS也就瘫了，因此非常需要有HA机制。
    * HDFS采取的方案是: `主备双活NameNode + Zookeeper集群(Master选举) + Journal(共享存储)`。
* **文件数据方面**
    * 数据通过`replicas`冗余来保证HA。


更详细的信息可以参考文章[HDFS的HA机制][2]。

### 主备NameNode + 自动主备切换
> HDFS也可以通过手动切换主备，本文主要关注通过`ZK进行辅助Master选举`的方式进行主备切换。

#### 建锁结点
当NameNode节点需要申请成为主结点时，需要通过ZK进行Master选举时，通过抢占在ZK里建立对应的锁结点。建立锁结点成功，那么说明选主成功。其中锁结点信息包括两部分:
* **临时结点**: `/hadoop-ha/${dfs.nameservices}/ActiveStandbyElectorLock` 
    * 如果ZK在一定的时间内收到不到对应的NameNode的心跳，会将这个临时结点删掉。
* **持久结点**: `/hadoop-ha/${dfs.nameservices}/ActiveBreadCrumb`  
    * 持久结点会在成为主结点是同时创建。建立持久结点的目的是为了NameNode和ZK之间通信假死带来脑裂问题。持久结点里会记录NameNode的地址。当发生脑裂时，下一个被选为主结点的NameNode会去查看是不是存在持久结点，如果存在，就会采取Fencing的措施，来防止脑裂问题。具体的Fencing方法有:
        * 通过调用旧的Active NameNode的HAServiceProtocolRPC来去transition旧的NameNode为StandBy状态。
        * 通过SSH方式登录到对应的NameNode机器上Kill掉对应的进程。
        * 执行用户自定义的Shell脚本去隔离进程。

#### 注册Watch监听
当NameNode申请成为主结点失败时，会向ZK注册一个监听事件，来监听对应的锁节点的目录变化，当然主要监听的是NodeDelete事件，会用来触发自动主备切换事件。

#### 自动主备切换
NameNode的自动主备切换主要由`ZKFailoverController`, `HealthMontior`和`ActiveStandbyElector`这3个组件来协同实现。
* `ZKFailoverController`启动时会创建HealthMonitor和ActiveStandbyElector两个组件，并向这两个组件注册对应的回调方法。
* `HealthMonitor`主要是用来监控NameNode的健康状态，如果检测到有状态变化，会调用回调函数来通知ZKFailoverController进行自动的主备选举。
* `ActiveStandbyElector`主要是负责和ZK交互, 里面封装了ZK相关的处理逻辑，当ZK master选举完成，会回调ZKFailoverController的相应方法来进行NameNode的主备状态切换。

具体的主备切换流程如下(可参考下面的HA图):
![HDFS-HA](/img/hadoop/HDFS_HA.png)
* **Active NameNode**  
① Active NameNode上的HealthMonitor发现NameNode上状态发生变化，比如没有响应。  
② 通过回调ZKFailoverController函数通知。  
③ ZKFailoverController收到通知后会调用ActiveStandbyElector去删除掉在ZK集群上创建的锁结点。对于正常情况下关闭的Active NameNode，也会将持久锁结点一并删除。  
④ ActiveStandbyElector call ZK集群删除对应的锁结点。  
⑤ 当删除结点成功后，AcitveStandbyElector会回调ZKFailoverController方法进行通知。  
⑥ ZKFailoverController会去将Active NameNode的状态切换为Standby。  

* **Standby NameNode**  
① Standby NameNode在第一次主备竞选时在ZK建立锁结点失败时会注册Watch监听。  
② 当Active NameNode进行主备切换删除锁结点，NodeDelete的事件触发Standby NameNode的ActiveStandByElector的自动创建锁结点，申请成为主结点的动作。  
③ 当申请主结点被ZK通过后，会回调ZKFailoverController进行NameNode的状态切换。  
④ ZKFailoverController调NameNode方法将状态从Standby更新为Active。  
⑤ NameNode从Journal集群里Sync最新元数据EditLog信息。  
⑥ 当所有的元数据信息整体对齐后，此时的NameNode才会真正对外提供服务。  

以上是正常情况下的主备切换流程。当Active NameNode整个机器宕机，或者和ZK失去通信后，根据ZK临时节点的特性，锁节点也会自动删除，自动触发主备切换。

#### 脑裂和Fencing
Zookeeper的工程实践里会经常出现“假死”的情况，即客户端到服务端的心跳不能正常发出，通讯出现问题。这样当超时超过设置的Session Timeout参数时，Zookeeper就会认为客户端已经挂掉了，会自动关闭session，删除锁节点，从而引发分布式系统里的双主或者脑裂的情况。比如HDFS里，会触发自动的主备切换，而实际上原来的Active NameNode还是好的，这样就存在两个Active NameNode在工作。

HDFS HA里解决脑裂问题就是在ZK里建立持久结点通过Fencing机制，可以阅读[持久结点](#建锁结点)。
具体到主备切换机制里，当Standby结点在② 时，会发现ZK上存在永久锁结点，那就会采取Fencing机制。当成功将原来的Active NameNode隔离（Kill或者进程隔离等），才会真正去call ZKFaioverController进行状态切换。

### Journal共享存储元数据
* Active NameNode向Journal集群结点同步写EditLog元数据，具体可参考[元数据的高并发修改](#元数据的高并发修改)部分。
* 而Standby NameNode则是定时从Journal集群同步EditLog元数据到本地。
* 在发生NameNode主备切换的时候，需要将Standby的NameNode的元数据同Journal集群结点的信息完全对齐后才可对外提供数据。

> Journal本身也是分布集群来通过Paxos算法来提供分布式数据一致性的保障。只有多数据结点通过投票以后才认为真正的数据写成功。

### 元数据保护
* 可以通过维护多份FSImage(落盘) + EditLog 副本来防止元数据损坏。


## HDFS的数据一致性
### 元数据一致性
* 主备双活NameNode之间的元数据
    * 通过Journal共享存储EditLog，每次切换主备时只有对齐EditLog以后才能对外提供服务。
* 内存与磁盘里元数据
    * `内存里的数据 = 最新的FSImage + EditLog`
        * 当有元数据修改时，往内存写时，需要先往EditLog里记录元数据的操作记录。
        * 当EditLog数据满了以后，会将EditLog应用FSImage里并和内存里的数据做同步，生成新的FSImage，清空EditLog。

### 数据一致性
HDFS会对写入的所有数据计算校验和（checksum），并在读取数据时验证。
* 写入的时候会往DataNode发Checksum值，最后一个写的DataNode会负责检查所有负责写的DataNode的数据正确性。
* 读数据的时候，客户端也会去和存储在DataNode中的校验和进行比较。

## HDFS高并发
### 元数据的高并发修改
主要的流程图如下:
![HDFS-HC](/img/hadoop/HDFS_High_Concurrency.png)
参考[博文](https://juejin.cn/post/6844903713966915598)

**主要的过程**:  
当有多个线程排除申请修改元数据时，会需要经过两阶段的对元数据资源申请加锁的过程。
* 第一次申请锁成功的线程，会首先生成`全局唯一且递增`的txid来作为这次元数据的标识，将元数据修改的信息（EditLog的transaction信息）写入到当下其中一个Buffer里（没有担任刷数据到磁盘的角色的Buffer里）。然后第一次快速释放锁。
* 此时前一步中的线程接着发起第二次加锁请求:
    * 如果请求失败（比如现在正在有其他的线程正在写Buffer）会将自己休眠1s然后再发起新的加锁请求。
    * 如果第二次请求加锁成功，会先check是否有线程正在进行刷磁盘的操作:
        * 如果是，那么就快速释放第二次加锁然后再把自己休眠等待下次加锁请求(因为已经有人在刷磁盘了，为了不阻塞其他线程写Buffer，先释放锁信息)。
        * 如果不是，那么会接着check是否自己的EditLog信息已经由在后面的其他线程刷进磁盘里:
            * 如果是，那么就直接释放第二次加锁请求直接线程退出，因为不再需要它做任何事情；
            * 如果还没刷进去，那么就由该线程担任起切换Buffer并刷数据到磁盘和Journal集群结点的重任。在切换Buffer以后，该线程会进行第二次释放锁的动作，这样其他线程可以继续往切换后的Buffer写数据了。在慢慢刷数据到本地磁盘或者通过网络刷数据到Journal结点的过程中，不会阻塞其他线程同时的写请求，提高并发量。


**主要的方法**:
* `分段加锁机制 + 内存双缓冲机制`
    * 分段加锁是指:
        * 第一阶段是在写内存缓冲区的申请对修改加锁。
        * 第二段是在申请刷缓冲区的数据到磁盘、Journal集群资格的时候申请加锁。
        * 整个过程中只有一个锁，保护的元数据的资源。当开始刷数据时，会立刻释放锁，不会阻塞后续其他往内存缓冲区写数据的线程。
    * 内存双缓存：
        * 缓冲1用来当下的写入Log。
        * 缓冲2用来读取已经写入的刷到磁盘和Journal结点。
        * 两个缓存会交换角色(需要时机判断)
* 缓冲数据批量刷`磁盘+网络`优化
* 多线程并发吞吐量支持



## Reference
* [Hadoop文档][1]
* [HDFS Design文档][2]
* [HDFS的HA机制][3]

<b><font color="red">本网站的文章除非特别声明，全部都是原创。
原创文章版权归数据元素</font>(</b>[DataElement](https://www.dataelement.top)<b><font color="red">)所有，未经许可不得转载!</font></b>  
**了解更多大数据相关分享，可关注微信公众号"`数据元素`"**
![数据元素微信公众号](/img/dataelement.gif)

[1]: https://hadoop.apache.org/docs/current/ 
[2]: https://hadoop.apache.org/docs/current/hadoop-project-dist/hadoop-hdfs/HdfsDesign.html
[3]: https://developer.ibm.com/zh/articles/os-cn-hadoop-name-node

