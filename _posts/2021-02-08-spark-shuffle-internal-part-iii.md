---
layout: post
title: "Spark Shuffle 内部机制（三）"
subtitle: "Spark Shuflle的前世今生"
date: 2021-02-08
author: "Hanke"
header-style: "text"
tags: [Spark, 大数据，大数据技术，开源]
---
> 在上两篇文章[Spark Shuffle 内部机制（一)][1]和[Spark Shuffle 内部机制（二)][2]中我们分别介绍了Spark Shuffle Write和Read的框架设计，在本篇中我们继续总结一下Spark Shuffle整个的发展历史。

## Spark Shuffle的前世今生
Spark的Shuffle在Write和Read两阶段有今天灵活的框架设计也是经过一步步不断完善和努力的结果，我们一起来回顾一下它的前世今生。  
![Spark-Shuffle-History](/img/spark/shuffle/Shuffle_History.png)

### 1) Spark 0.8 之前HashShuffleWriter
在Spark 0.8以前用的是basic的HashShuffleWriter，整体的实现就是Shuffle Write框架里介绍是[仅需要map，不需要combine和sort的场景][1]。实现上比较简单直接，每个map task按照下游reduce tasks个数即reduce分区个数，每个分区生成一个文件写入磁盘。如果有M个map tasks, R个reduce tasks，那么就会产生`M * R`个磁盘文件。因此对于大分区情况非常不友好，会生成大量的碎文件，造成I/O性能下降，文件连接数过大，导致resource吃紧，进而影响整体性能。具体可参考下图:
![HashBasedShuffleWriter](/img/spark/shuffle/HashBasedShuffle.png)

### 2) Spark 0.8.1 引入文件Consolidation的HashShuffleWriter
由于basic的HashShuffleWriter生成的碎小文件过多，为了解决这个问题引入了文件Consolidation机制。在同一个core上运行的所有的map tasks对应的相同的分区数据会写到相同的buffer里最终对应分区的一个分区文件。如果有M个map tasks, R个reduce tasks，C个cores，那么最终会产生`C * R`个磁盘文件。如果C比M小，那么对比basic的HashShuffleWriter，文件个数有所下降，性能会得到提升。具体过程可参照下图:
![Consolidation-HashBasedShuffleWriter](/img/spark/shuffle/Consolidation_HashShuffleWriter.png)


### 3) Spark 1.1 引入SortShuffleWriter
虽然Consolidation的机制在一定程度上减少文件个数，但是当cores和reduce的task过多的时候一个map task依然会产生大量的文件。在Spark 1.1里首次引入了基于sort的shuffle writer，整体的实现是Shuffle Writer框架里介绍的[需要map，需要sort，不需要combine的场景][1]。每个map task的输出数据会按照partitionId排序，最终一个map task只会输出一个分区文件包括这个map task里的所有分区数据 + 分区索引文件供下游shuffle read使用，大大减少了文件个数。具体过程可参照下图: 
![SortShuffleWriter](/img/spark/shuffle/SortShuffleWriter.png)

### 4) BypassMergeSortShuffleWriter
SortShuffleWriter的引入大大减少了文件个数，但是也额外增加了按partitionId排序的操作，加大了时延。对于分区个数不是太大的场景，简单直接的HashShuffleWriter还是有可借鉴之处的。BypassMergeSortShuffleWriter融合了HashShuffleWriter和SortShuffleWriter的优势，每个map task首先按照下游reduce tasks的个数，生成对应的分区数据和分区文件(每一个分区对应一个分区文件)，在最终提供给下游shuffle read之前，会将map task产生的这些中间分区文件做一个合并(Consolidation)，只输出一个分区文件包含所有的分区数据 + 分区索引文件供下游shuffle read使用。具体过程可参照下图: 
![BypassMergeSortShuffleWriter](/img/spark/shuffle/BypassMergeSortShuffleWriter.png)
> 需要注意的是BypassMergeSortShuffleWriter不适合分区比较大的场景，因为在Shuffle Writer阶段，一个map task会为每个分区开一个对应的buffer，如果分区过大，那么占用的内存比较大，性能也会有影响。具体可以参照Spark Shuffle Writer 框架里[仅需要map，不需要combine和sort的场景][1]的解释，这里不再赘述。

### 5) Spark 1.4 引入UnsafeShuffleWriter
UnsafeShuffleWriter是一种Serialized Shuffle，主要是对于map里不需要聚合和排序但是partition个数较多的情况下一种优化。在[Shuffle Writer框架里需要map需要sort的场景][1]中提到对于这种场景，用的是数组结构，存放的是普通的record的Java对象。当record比较大时，非常占用内存，也会导致GC频繁。Serialized Shuffle将record序列化以后放在内存，进一步减少内存的占用、降低GC频率。具体可参考下图和前篇关于Shuffle优化部分Serialized Shuffle的介绍:
![Map-No-Sort-No-Combine-Serialized-Shuffle-Write](/img/spark/shuffle/Serialized_Shuffle_in_Shuffle_Write.png)

### 6) 今天的Spark Shuffle
在Spark 2.0里，第一版的HashShuffleWriter彻底退出历史舞台。今天的Spark Shuffle Writer只有三种writer方式:
* Sort
    * SortShuffleWriter(Default)
    * UnsafeShuffleWriter
        * 也叫Tungsten-sort
* BypassMergeSortShuffleWriter

#### 是否需要Sort？
默认模式下用的是SortShuffleWriter的方式，但用户也可以通过指定的方式来选择更适合的Shuffle方式。  
* 如果分区个数不超过BypassMergeSort的阈值`spark.shuffle.sort.bypassMergeThreshold`，就用`BypassMergeSortShuffleWriter`。
* 否则就用Sort的方式。  

样例代码参见: https://github.com/apache/spark/blob/master/core/src/main/scala/org/apache/spark/shuffle/sort/SortShuffleManager.scala
```scala
override def registerShuffle[K, V, C](
    shuffleId: Int,
    numMaps: Int,
    dependency: ShuffleDependency[K, V, C]): ShuffleHandle = {
  if (SortShuffleWriter.shouldBypassMergeSort(conf, dependency)) {
    // If there are fewer than spark.shuffle.sort.bypassMergeThreshold partitions and we don't
    // need map-side aggregation, then write numPartitions files directly and just concatenate
    // them at the end. This avoids doing serialization and deserialization twice to merge
    // together the spilled files, which would happen with the normal code path. The downside is
    // having multiple files open at a time and thus more memory allocated to buffers.
    new BypassMergeSortShuffleHandle[K, V](
      shuffleId, numMaps, dependency.asInstanceOf[ShuffleDependency[K, V, V]])
  } else if (SortShuffleManager.canUseSerializedShuffle(dependency)) {
    // Otherwise, try to buffer map outputs in a serialized form, since this is more efficient:
    new SerializedShuffleHandle[K, V](
      shuffleId, numMaps, dependency.asInstanceOf[ShuffleDependency[K, V, V]])
  } else {
    // Otherwise, buffer map outputs in a deserialized form:
    new BaseShuffleHandle(shuffleId, numMaps, dependency)
  }
}
``` 
#### 用哪种Sort方式？
可以通过`spark.shuffle.manager`来设置SortShuffleManager，默认是用的普通的sort方式。如果需要用序列化的sort方式进行优化的话，可以将该参数设置成`tungsten-sort`即可。
```scala
    // Let the user specify short names for shuffle managers
    val shortShuffleMgrNames = Map(
      "sort" -> classOf[org.apache.spark.shuffle.sort.SortShuffleManager].getName,
      "tungsten-sort" -> classOf[org.apache.spark.shuffle.sort.SortShuffleManager].getName)
    val shuffleMgrName = conf.get(config.SHUFFLE_MANAGER)
    val shuffleMgrClass =
      shortShuffleMgrNames.getOrElse(shuffleMgrName.toLowerCase(Locale.ROOT), shuffleMgrName)
    val shuffleManager = instantiateClass[ShuffleManager](shuffleMgrClass)

```

至此，关于Spark Shuffle相关的内部机制和历史都基本介绍完毕。具体使用哪种Shuffle的方式还需要根据场景实际需求来进行进一步的调优和配置，希望以上几篇文章能对大家有所启发。敬请关注本公众号后续数据相关技术分享。

## Reference
[Spark 官方文档](https://spark.apache.org/docs/latest/rdd-programming-guide.html#shuffle-operations)  
《大数据处理框架Spark的设计与实现》  
[Hadoop之MapReduce内部机制](https://dataelement.top/2021/01/04/MapReduce-in-Hadoop/)   
[Spark Shuffle 内部机制（一)][1]  
[Spark Shuffle 内部机制（二)][2]  

[1]:https://dataelement.top/2021/02/03/spark-shuffle-internal-part-i
[2]:https://dataelement.top/2021/02/05/spark-shuffle-internal-part-ii

<b><font color="red">本网站的文章除非特别声明，全部都是原创。
原创文章版权归数据元素</font>(</b>[DataElement](https://www.dataelement.top)<b><font color="red">)所有，未经许可不得转载!</font></b>  
**了解更多大数据相关分享，可关注微信公众号"`数据元素`"**
![数据元素微信公众号](/img/dataelement.gif)
