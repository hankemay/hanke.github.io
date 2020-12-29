---
layout: post
title: "揭秘Apache Spark 3.0 新特性在Freewheel核心业务数据团队的应用与实战"
subtitle: "新特性与特性"
date: 2020-12-29
author: "Hanke"
header-style: "text"
tags: [Spark]
---
### 引言
相信作为Spark的粉丝或者平时工作跟Spark相关的同学大多知道，Spark 3.0在2020年6月官方重磅发布，并于9月发布稳定线上版本，这是Spark有史以来最大的一次release，共包含了3400多个patch，而且恰逢Spark发布的第十年，具有非常重大的意义。  

我们团队在Spark发布后，快速自己搭好Spark 3.0的裸机集群并在其上进行了初步的调研，发现相较于Spark 2.x 确实有性能上的提升。于是跟AWS EMR和Support团队进行了多次沟通表达我们的迫切需求后，EMR团队给予了快速的响应，在11月底发布了内测版本。作为第一批内测用户，我们做了Data Pipelines上各个模块的升级，测试和数据验证。团队通过高效的敏捷开发顺利赶在2020年圣诞广告季之前在生产环境发布上线，整体**性能提升高达40%**（对于大batch）的数据，**AWS Cost平均节省25%~30%之间**，大约每年至少能为公司节省16万美元。目前线上稳定运行，借助此次升级能够更从容为Freewheel高速增长业务量和数据分析需求保驾护航。  

本篇文章主要是和大家分享一下Spark 3.0在Freewheel大数据团队升级背后的故(血)事(泪)和相关的实战经验，希望能对大家以后的使用Spark3.0特别是基于AWS EMR上开发有所帮助，可以少走一些弯路。

### Spark 3.0 关键新特性回顾
Spark 3.0官方的[Release Notes](Spark Release Notes)可以看到，这次大版本的升级主要是集中在性能化和文档丰富上，其中46%的优化都集中在Spark SQL上。今天Spark SQL的优化不仅仅服务于SQL语言，还服务于机器学习、流计算和DataFrame等计算任务， 因此社区对于Spark SQL的投入非常大。对外公布的TPC-DS性能测试结果相较于Spark 2.4会有2倍的提升。SQL优化里最引人注意的非[`Adaptive Query Engine`](AQE)莫属了， 还有一些其他的优化比如[Dynamic Pruning Partition](DPP)，通过Aggregator注册[UDAF](Registor Aggregator to UDAF)(User Define Aggregator Function)等都极大的提升了SQL引擎的性能。 

#### Adaptive Query Engine (AQE)

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

