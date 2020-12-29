---
layout: post
title: "揭秘Apache Spark 3.0 新特性在Freewheel核心业务数据团队的应用与实战"
subtitle: "新特性"
date: 2020-12-29
author: "Hanke"
header-style: "text"
tags: [Spark]
---
### 引言
相信作为Spark的粉丝或者平时工作跟Spark相关的同学大多知道，Spark 3.0在2020年6月官方重磅发布，并于9月发布稳定线上版本，这是Spark有史以来最大的一次release，共包含了3400多个patch，而且恰逢Spark发布的第十年，具有非常重大的意义。  

我们团队在Spark发布后，快速自己搭好Spark 3.0的裸机集群并在其上进行了初步的调研，发现相较于Spark 2.x 确实有性能上的提升。于是跟AWS EMR和Support团队进行了多次沟通表达我们的迫切需求后，EMR团队给予了快速的响应，在11月底发布了内测版本。作为第一批内测用户，我们做了Data Pipelines上各个模块的升级，测试和数据验证。团队通过高效的敏捷开发顺利赶在2020年圣诞广告季之前在生产环境发布上线，整体**性能提升高达40%**（对于大batch）的数据，**AWS Cost平均节省25%~30%之间**，大约每年至少能为公司节省一百多万人民币(约16万美元)。目前线上稳定运行，借助此次升级能够更从容为Freewheel高速增长业务量和数据分析需求保驾护航。  

本篇文章主要是和大家分享一下Spark 3.0在Freewheel大数据团队升级背后的故(血)事(泪)和相关的实战经验，希望能对大家以后的使用Spark3.0特别是基于AWS EMR上开发有所帮助，可以少走一些弯路。

### Spark 3.0 关键新特性回顾
Spark 3.0官方的[Release Notes](Spark Release Notes)可以看到，这次大版本的升级主要是集中在性能优化和文档丰富上(如下图)，其中46%的优化都集中在Spark SQL上。  
<img src="/img/spark3-0.png" width="70%" height="70%">

今天Spark SQL的优化不仅仅服务于SQL语言，还服务于机器学习、流计算和DataFrame等计算任务， 因此社区对于Spark SQL的投入非常大。对外公布的TPC-DS性能测试结果相较于Spark 2.4会有2倍的提升。SQL优化里最引人注意的非[`Adaptive Query Execution`](AQE)莫属了， 还有一些其他的优化比如[Dynamic Pruning Partition](DPP)，通过Aggregator注册[UDAF](Registor Aggregator to UDAF)(User Define Aggregator Function)等等都极大的提升了SQL引擎的性能。  
本文会着重回顾AQE新特性及相关值得注意的文档方面的变化。其他更多的信息感兴趣的同学可以参考官网notes.  

#### Adaptive Query Execution (AQE)
AQE对于整体的Spark SQL的执行过程做了相应的调整和优化(如下图)，它最大的亮点是可以根据已经完成的计划结点`真实且精确的执行统计结果`来不停的`反馈并重新优化`剩下的执行计划。

![AQE](/img/AQE.gif)
Spark 2.x的SQL执行过程:
* 当用户提交了Spark SQL/Dataset/DataFrame时，在逻辑执行计划阶段，Spark SQL的Parser会用ANLTER将其转化为对应的语法树（Unresolved Logic Plan)，接着Analyzer会利用catalog里的信息找到表和数据类型及对应的列将其转化为解析后有schema的Logical Plan，然后Optimizer会通过一系列的优化rule进行算子下推（比如filter， 列剪裁)，提前计算常量（比如当前时间），replace一些操作符等等来去优化Logical Plan。  
* 而在物理计划阶段，Spark Planner会将各种物理计划策略作用于对应的Logical Plan节点上，生成多个物理计划，然后通过CBO选择一个最佳的作为最终的物理算子树(比如选择用Broadcast的算子，而不是SortMerge的Join算子)， 最终将算子树的节点转化为Spark底层的RDD,Transformation和Action等，以支持其提交执行。 
在Spark 3.0之前， Spark的Catalyst的优化主要是通过基于逻辑计划的rule和物理计划里的[CBO](CBO)，这些优化要么基于数据库里的`静态`信息，要么通过预先得到统计信息， 比如数值分布的直方图等来`预估`并判断应该使用哪种优化策略。这样的优化存在很多问题，比如数据的meta信息不准确或者不全，或者复杂的filter，黑盒的UDFs等导致无法预估正确的数据量，因此很难得到较优的优化策略。  

**主要功能点**  
此时，提出AQE通过`真实且精确的执行统计结果`进行优化就很有意义了。基于这个设计和背景，AQE就能够比较方便解决用户在使用Spark中一些头疼的地方。主要体现在以下三个方面：
* **自动调整reducer的数量，减小partition数量**  
    * Spark任务的并行度一直是让用户比较困扰的地方。如果并行度太大的话，会导致task 过多，overhead比较大，整体拉慢任务的运行。而如果并行度太小的，数据分区会比较大，容易出现OOM的问题，并且资源也得不到合理的利用。而且由于Spark Context整个任务的并行度，需要一开始设定好且没法动态修改，这就很容易出现任务刚开始的时候数据量大需要大的并行度，而运行的过程中通过转化过滤可能最终的数据集已经变得很小，最初设定的分区数就过大了。AQE能够很好的解决这个问题，在reducer去读取数据时，会根据用户设定的分区数据的大小来自动调整和合并(`Coalesce`)小的partition, 自适应地减小partition的数量，以减少资源浪费和overhead，提升任务的性能。参考示例图:
![AQE](/img/AQE-Coalesce.png)

* **自动解决Join时的数据倾斜问题**  
    * Join里如果出现某个key的数据倾斜问题，那么基于上就是这个任务的性能杀手了。在AQE之前，用户没法自动处理Join中遇到的这个棘手问题，需要借助外部手动收集数据统计信息，并做额外的加盐，分批处理数据等相对繁琐的方法来应对数据倾斜问题。而AQE由于可以实时拿到运行时的数据，通过`Skew Shuffle Reader`自动调整不同key的数据来避免数据倾斜，从而提高性能。参考示例图:
![AQE](/img/AQE-DataSkew.png)

* **优化Join策略**
    * AQE可以在Join的初始阶段获悉数据的输入特性，并基于此选择适合的Join算法从而最大化地优化性能。比如从Cost比较高的SortMerge在不超过阈值的情况下调整为代价较小的Broadcast Join。参考示例图:
![AQE](/img/AQE-JoinSwitch.png)

#### 文档

### 团队Data Pipelines简介

### 实践成果

### 我们做了什么？

### 为什么既能提升性能又能省钱？

### 踩了什么坑？

### 未来展望

### Reference
* [Spark 3.0 Release Notes](https://spark.apache.org/releases/spark-release-3-0-0.html)
* [AQE](https://databricks.com/blog/2020/05/29/adaptive-query-execution-speeding-up-spark-sql-at-runtime.html)
* [DPP](https://blog.knoldus.com/dynamic-partition-pruning-in-spark-3-0/)
* [UDAF](https://issues.apache.org/jira/browse/SPARK-27296)
* [CBO](https://databricks.com/blog/2017/08/31/cost-based-optimizer-in-apache-spark-2-2.html)

