---
layout: post
title: "Spark Shuffle Internal"
subtitle: "Internal Shuffle Framework and Design"
date: 2021-01-18
author: "Hanke"
header-style: "text"
tags: [Spark, Open Source ]
---
## Spark Shuffle 是什么
Spark Shuffle是根据数据处理需求将数据按着某种方式重新混洗后，以便于后面的数据处理。比如ReduceByKey()的操作，通过先将数据按照key进行重新分区以后，然后对每个key的数据进行reduce操作。  

Spark Shuffle共包括两部分:
* **Spark Shuffle Write**
    * 解决上游输出数据的分区问题
* **Spark Shuffle Read**
    * 通过网络拉取对应的分区数据，重新组织，然后为后续的操作提供输入数据

## Spark Shuffle 要解决的问题
* 如何灵活支持不同计算的场景下不同的需求？排序、聚合、分区？
    * 通过灵活的框架设计来满足不同的需求
* 如何应对大数据量的情况下面临的内存压力？
    * 通过`内存 + 磁盘 + 数据结构设计`来解决内存问题
* 如何保证Shuffle过程的高性能问题？
    * 减少网络传输
        * Map端的Reduce
    * 减少碎小文件
        * 按PartitionId进行排序并合并多个碎文件为一个，然后加上分区文件索引供下游使用

## Spark Shuffle的框架

## Spark Shuffle的发展历史


<b><font color="red">本网站的文章除非特别声明，全部都是原创。
原创文章版权归数据元素</font>(</b>[DataElement](https://www.dataelement.top)<b><font color="red">)所有，未经许可不得转载!</font></b>  
**了解更多大数据相关分享，可关注微信公众号"`数据元素`"**
![数据元素微信公众号](/img/dataelement.gif)
