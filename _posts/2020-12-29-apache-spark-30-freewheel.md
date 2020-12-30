---
layout: post
title: "揭秘Apache Spark 3.0 新特性在Freewheel核心业务数据团队的应用与实战"
subtitle: "新特性"
date: 2020-12-29
author: "Hanke"
header-style: "text"
tags: [Spark]
---
## 引言
相信作为Spark的粉丝或者平时工作跟Spark相关的同学大多知道，Spark 3.0在2020年6月官方重磅发布，并于9月发布稳定线上版本，这是Spark有史以来最大的一次release，共包含了3400多个patch，而且恰逢Spark发布的第十年，具有非常重大的意义。  

我们团队在Spark发布后，快速自己搭好Spark 3.0的裸机集群并在其上进行了初步的调研，发现相较于Spark 2.x 确实有性能上的提升。于是跟AWS EMR和Support团队进行了多次沟通表达我们的迫切需求后，EMR团队给予了快速的响应，在11月底发布了内测版本。作为第一批内测用户，我们做了Data Pipelines上各个模块的升级，测试和数据验证。团队通过高效的敏捷开发赶在2020年圣诞广告季之前在生产环境顺利发布上线，整体**性能提升高达40%**（对于大batch）的数据，**AWS Cost平均节省25%~30%之间**，大约每年至少能为公司节省一百多万人民币(约16万美元)。目前线上稳定运行，借助此次升级能够更从容为Freewheel高速增长业务量和数据分析需求保驾护航。  

本篇文章主要是和大家分享一下Spark 3.0在Freewheel大数据团队升级背后的故(血)事(泪)和相关的实战经验，希望能对大家以后的使用Spark3.0特别是基于AWS EMR上开发有所帮助，可以在Spark升级的道路上走的更顺一些。

## 团队简介
Freewheel核心业务数据团队的主要工作是通过收集，分析来自用户的视频广告数据，来帮助客户更好地制定广告计划，满足客户不断增长的业务需求，最终帮助客户实现业务的增长。其中最主要的两类数据分别是预测数据和历史数据:
* **预测数据**会根据用户历史广告投放情况进行算法分析和学习来得到未来预测情况，在此基础上向客户提供有价值的数据分析结果，比如广告投放是否健康，广告位是否足够，当前的广告售卖是否合理等等信息。通过这些数据分析的反馈可以帮助用户更好地在广告定价、售期等方面做正确的决定，最终达到自己的销售目标。
* **历史数据**主要是提供用户业务场景数据分析所需要的功能，比如数据查询，Billing账单，广告投放情况，市场策略等, 并且通过大量的历史数据从多维度多指标的角度提供强有力的BI分析能力去帮助用户洞察数据发生的变化，发现潜在的问题和市场机会。  

作为核心业务数据团队里重要的成员，**Transformer**团队的主要负责:
* **基于大数据平台技术建立Data Pipelines**
    * 负责将交易级别的数据转化为分析级别的数据服务下游所有的数据产品
* **构建统一的数据仓库**
    * 通过分层业务模型来构建所有数据产品不同场景下（历史或者预测）使用的一致的业务视图和指标
    * 提供不同粒度或者维度的聚合事实数据
    * 提供基于特定场景的数据集市
* **提供统一的数据发布服务和接口**

### 数据建模和Pipelines架构
当交易级别的广告数据（历史或者预测）数据进入系统后，会通过数据建模和Pipelines进行统一的建模或者分析，视业务需要更进一步构建数据集市，生成的聚合事实数据会被发布到数据仓库Hive和Clickhouse里供下游所有的数据产品通过Presto或者Clickhouse查询引擎来消费。如下是整体建模和Pipelines的架构图：
![Team Intro](/img/team-intro.png)

其中主要模块包括：
* **Optimus**  
    * 正如它的名字一样，`Optimus`同样是Transformer团队的模块中的领袖人物，肩负业务数据团队最重要的数据建模部分。通过分层数据建模的方式来构建统一的基于的上下文数据模型，保障所有下游产品在不同的应用和业务场景下的计算指标，计算逻辑一致。比如预测数据和历史数据同样的指标含义，就使得提供给客户的数据对比更有说服力和决策指导意义。目前它会产生将近四十张左右的小时粒度的历史事实表和预测事实表。目前每天处理的数据在TB级别，会根据每个小时的数据量自动进行扩或者缩集群，保证任务的高性能同时达到资源的高效利用。
* **JetFire**  
    * `JetFire`是一个基于Spark的通用ETL框架，支持用户通过SQL或者Code的方式灵活的定制ETL任务处理和分析数据。目前主要用于Post-Optimus的场景，生成基于特定业务场景更高聚合粒度的数据集市上。比如生成`todate`(迄今为止)的统计指标，比如每个客户截止到目前或者过去18个月的广告投放总数。这样就可以避免每次查询对底层数据或者Optimus生成的聚合数据进行全扫。生成一次供多次查询，可以极大提高查询效率，降低成本。
* **New Publisher**
    * 基于Spark的数据发布模块，负责将数据发布到数据仓库里。由于数据建模产生的数据按日期进行分区，当存在Late Data的时候，很容易生成碎小文件，Publisher通过发布数据前合并碎小文件的功能来提升下游的查询效率。
* **Bumblebee**
    * 主要是为数据建模和Pipelines的各个模块提供模块测试和集成测试环境，供业务开发的同学使用。此外，基于此提供所有数据Pipelines的整体一致CD和灾备方案，保障在极端场景下系统的快速启动和恢复。 
* **Data Restatement**
    * 除了日常的数据Pipelines，在客户数据投放出现问题或者数据仓库数据出现偏差遗漏时，需要自动的修数据Pipelines来支持大范围的数据修正和补偿。整体的作业调度需要保证日常工作正常完成的情况下，尽快完成数据修正工作。目前提供整个batch或者delta两种方式修数据，来满足不同的应用场景。
* **Transformer API**
    * 负责为下游提供数据发布信息，来触发一些订阅的报表或者产品发布。  

除了Transformer API服务部署在EKS上，其他相关模块目前都运行在AWS EMR上。团队基于以上的模块为公司的业务发展提供有力的数据和技术保障。

## 实践成果
这次升级将上面的主要实践成果如下：
### 性能提升明显
* **历史数据**Pipeline对于大batch的数据（300~400G/每小时）性能`提升高达40%`， 对于小batch（小于100G/每小时）提升效果没有大batch提升的那么明显，每天所有batches`平均提升水平27.5%`左右。
* **预测数据**性能提升平均`30%` 
    * 由于输入源不一样，目前是分别两个pipeline在跑历史和预测数据，产生的表的数目也不太一样，因此做了分别的评估。  

以历史数据上线后的端到端到运行时间为例（如下图）, 肉眼可见整体pipeline的运行时间有了明显的下降，能够更快的输出数据供下游使用。  
![runtime](/img/runtime.png)

### 集群内存使用降低
集群内存使用对于大batch达`30%`左右，每天平均`平均节省25%`左右。  
以历史数据上线后的运行时集群的memory在ganglia上的截图为例（如下图）, 整体集群的内存使用从41.2T降到30.1T，这意味着我们可以用更少的机器花更少的钱来跑同样的Spark任务。  
![memory](/img/spark-3-memory.png)

### AWS Cost降低
Pipelines做了自动的Scale In/Scale Out, 在需要资源的时候扩集群的task结点，在任务结束后自动去缩集群的task结点，且会根据每次batch数据的大小通过算法学习得到最佳的机器数。通过升级到Spark 3.0后，由于现在任务跑的更快并且需要的机器更少，上线后统计AWS Cost每天节省`30%`左右，大约一年能为公司`节省一百多万人民币`（约160K美元）。  

如下是历史数据Pipeline上线后，通过AWS Billing得到的Cost数据， 可能看到从每天平均1700美元降到稳定的1200美元左右。
![aws-cost](/img/spark-3-aws-cost.jpg)




在我们具体看具体做了什么并且背后有什么魔法在起作用之前，先一起回顾一下Spark 3.0这次发布里关键新特性。

## Spark 3.0 关键新特性回顾
从Spark 3.0官方的[Release Notes][1]可以看到，这次大版本的升级主要是集中在性能优化和文档丰富上(如下图)，其中46%的优化都集中在Spark SQL上。  
<img src="/img/spark3-0.png" width="70%" height="70%">

今天Spark SQL的优化不仅仅服务于SQL语言，还服务于机器学习、流计算和DataFrame等计算任务， 因此社区对于Spark SQL的投入非常大。对外公布的TPC-DS性能测试结果相较于Spark 2.4会有**2倍**的提升。SQL优化里最引人注意的非[`Adaptive Query Execution`][2]莫属了， 还有一些其他的优化比如[Dynamic Pruning Partition][3]，通过Aggregator注册[UDAF][4](User Define Aggregator Function)等等都极大的提升了SQL引擎的性能。  
本文会着重回顾AQE新特性及相关关注的特性和文档监控方面的变化。其他更多的信息比如复用子查询优化，SQL Hints，ANSI SQL兼容，SparkR向量化读写，加速器感知GPU调度等等，感兴趣的同学可以参考官网[notes][1].  

### Adaptive Query Execution (AQE)
AQE对于整体的Spark SQL的执行过程做了相应的调整和优化(如下图)，它最大的亮点是可以根据已经完成的计划结点`真实且精确的执行统计结果`来不停的`反馈并重新优化`剩下的执行计划。

![AQE](/img/AQE.gif)
Spark 2.x的SQL执行过程:
* 当用户提交了Spark SQL/Dataset/DataFrame时，在逻辑执行计划阶段，Spark SQL的Parser会用ANLTER将其转化为对应的语法树（Unresolved Logic Plan)，接着Analyzer会利用catalog里的信息找到表和数据类型及对应的列将其转化为解析后有schema的Logical Plan，然后Optimizer会通过一系列的优化rule进行算子下推（比如filter， 列剪裁)，提前计算常量（比如当前时间），replace一些操作符等等来去优化Logical Plan。  
* 而在物理计划阶段，Spark Planner会将各种物理计划策略作用于对应的Logical Plan节点上，生成多个物理计划，然后通过CBO选择一个最佳的作为最终的物理算子树(比如选择用Broadcast的算子，而不是SortMerge的Join算子)， 最终将算子树的节点转化为Spark底层的RDD,Transformation和Action等，以支持其提交执行。 
在Spark 3.0之前， Spark的Catalyst的优化主要是通过基于逻辑计划的rule和物理计划里的[CBO][5]，这些优化要么基于数据库里的`静态`信息，要么通过预先得到统计信息， 比如数值分布的直方图等来`预估`并判断应该使用哪种优化策略。这样的优化存在很多问题，比如数据的meta信息不准确或者不全，或者复杂的filter，黑盒的UDFs等导致无法预估正确的数据量，因此很难得到较优的优化策略。  

**主要功能点**  
此时，提出AQE通过`真实且精确的执行统计结果`进行优化就很有意义了。基于这个设计和背景，AQE就能够比较方便解决用户在使用Spark中一些头疼的地方。主要体现在以下三个方面：
* **自动调整reducer的数量，减小partition数量**  
    * Spark任务的并行度一直是让用户比较困扰的地方。如果并行度太大的话，会导致task 过多，overhead比较大，整体拉慢任务的运行。而如果并行度太小的，数据分区会比较大，容易出现OOM的问题，并且资源也得不到合理的利用。而且由于Spark Context整个任务的并行度，需要一开始设定好且没法动态修改，这就很容易出现任务刚开始的时候数据量大需要大的并行度，而运行的过程中通过转化过滤可能最终的数据集已经变得很小，最初设定的分区数就过大了。AQE能够很好的解决这个问题，在reducer去读取数据时，会根据用户设定的分区数据的大小(`spark.sql.adaptive.advisoryPartitionSizeInBytes`)来自动调整和合并(`Coalesce`)小的partition, 自适应地减小partition的数量，以减少资源浪费和overhead，提升任务的性能。参考示例图:
![AQE](/img/AQE-Coalesce.png)

* **自动解决Join时的数据倾斜问题**  
    * Join里如果出现某个key的数据倾斜问题，那么基于上就是这个任务的性能杀手了。在AQE之前，用户没法自动处理Join中遇到的这个棘手问题，需要借助外部手动收集数据统计信息，并做额外的加盐，分批处理数据等相对繁琐的方法来应对数据倾斜问题。而AQE由于可以实时拿到运行时的数据，通过`Skew Shuffle Reader`自动调整不同key的数据大小(`spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes`)来避免数据倾斜，从而提高性能。参考示例图:
![AQE](/img/AQE-DataSkew.png)
/img/shuffle2.png
* **优化Join策略**
    * AQE可以在Join的初始阶段获悉数据的输入特性，并基于此选择适合的Join算法从而最大化地优化性能。比如从Cost比较高的SortMerge在不超过阈值的情况下调整为代价较小的Broadcast Join。参考示例图:
![AQE](/img/AQE-JoinSwitch.png)

### Dynamic Pruning Partition(DPP)
DPP主要解决的是对于星型模型的查询场景中过滤条件无法下推的情况。通过DPP可以将小表过滤后的数据作为新的过滤条件下推到另一个大表里，从而可以做到对大表scan运行阶段的提前过滤掉不必要的partition读取。这样也可以避免引入不必要的额外ETL过程（提前生成另外过滤后的大表），在查询的过程中极大的提升查询性能， 感兴趣的同学可以更进一点阅读[DPP][3]的详细信息。

### 通过Aggragtor注册UDAF 
通过用户定制实现的Aggregator来注册实现UDAF，可以避免对每一行的数据反复进行序列化和反序列化来进行聚合，而只需在整个分区里序列化一次, ，缓解了对CPU的压力, 提升性能。假如一个DataFrame有100万行数据共10个paritions，那么旧的UDAF方式的序列化反序列化需要至少100万+10(合并分区里的结果)次。
![UDAF](/img/UDAF.png)
而新的函数只需要10次即可，大大减少整体的序列化操作。其中实现部分最主要的区别体现在UDAF的`update`函数部分：
```scala
//Old Way
def update(buf: MutableAggregationBuffer, input: Row): Unit = {
  val agg = buf.getAs[AggregatorType](0)  // UDT deserializes the aggregator from 'buf'
  agg.update(input)    // update the state of your aggregation
  buf(0) = agg    // UDT re-serializes the aggregator back into buf
}
//New way
def update(agg: AggregatorType, input: Row): AggregatorType = {
  agg.update(input) // update the state of your aggregator from the input
  agg // return the aggregator
}
```
更多技术细节部分可以阅读[Aggregator 注册UDAF](UDAF)。

### 文档与监控
Spark 3.0完善和丰富了很多文档及监控信息，来辅助大家更好的进行调优和监控任务的性能。  

#### Spark SQL 和 Web UI文档
增加了[Spark SQL语法][6]、[SQL配置的文档页面][7] 和相关[WebUI的文档][8]。

#### 更多的Shuffle 指标
Spark 3.0引入了更多可观察的指标来去观测数据的运行质量。Shuffle是Spark任务里非常重要的一部分，如果能拿到更详细的阶段数据，那么对于程序的调优是很有帮助的。
<img src="/img/shuffle2.png" width="70%" height="70%">

#### 新的Structured Streaming UI 
作为社区主推的Spark实时的模块Structured Streaming是在Spark 2.0中发布的，这次在Spark 3.0中正式加入了UI的配置。新的UI主要包括了两种统计信息，已完成的Streaming查询聚合的信息和正在进行的Streaming查询的当前信息， 具体包括Input Rate, Process Rate,Input Rows, Batch Duration和Operate Duration，可以辅助用户更进一步观察任务的负载和运行能力。
![Structured Streaming UI](/img/webui-structured-streaming-detail.png)

### 生态圈建设
扩展相关生态圏版本的升级和建设
* 支持Java 11
* 支持Hadoop 3
* 支持Hive 3


## 我们做了什么？

## 为什么既能提升性能又能省钱？

## 踩了什么坑？

## 未来展望

## Reference
* [Spark 3.0 Release Notes][1]
* [AQE][2]
* [DPP][3]
* [UDAF][4]
* [CBO][5]
* [Spark SQL语法][6]
* [Spark SQL配置][7]
* [Spark Web UI使用][8]

[1]: https://spark.apache.org/releases/spark-release-3-0-0.html
[2]: https://databricks.com/blog/2020/05/29/adaptive-query-execution-speeding-up-spark-sql-at-runtime.html
[3]: https://blog.knoldus.com/dynamic-partition-pruning-in-spark-3-0/
[4]: https://issues.apache.org/jira/browse/SPARK-27296
[5]: https://databricks.com/blog/2017/08/31/cost-based-optimizer-in-apache-spark-2-2.html
[6]: https://github.com/apache/spark/blob/master/docs/sql-ref-syntax-qry-select.md
[7]: https://spark.apache.org/docs/latest/configuration.html#spark-sql
[8]: https://github.com/apache/spark/blob/master/docs/web-ui.md


