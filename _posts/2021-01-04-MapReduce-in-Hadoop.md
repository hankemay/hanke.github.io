---
layout: post
title: "Hadoop之MapReduce内部机制"
subtitle: "MapReduce到底有什么问题？"
date: 2021-01-04
author: "Hanke"
header-style: "text"
tags: [Hadoop, MapReduce, BigData]
---
## Map Reduce的过程详细解析
![Map-Reduce-process](/img/hadoop/Hadoop-MapReduce.png)
① : 每个数据的Split对应一个Map任务作为Map的输入，一般来说是HDFS的一个Block。  
② : Map产生的数据会先写入到一个环形的内存的Buffer空间里。   
③ : 当Buffer满了以后, 会Spill溢出数据到磁盘里。在溢出之前会先按照Partition函数对数据进行分区(默认是取key的hash值然后根据Reducer的个数进行取模)，然后按照Key进行排序(快速排序)。如果设置了Combiner会在写入磁盘前，对数据进行Combine操作，通过减少key的数据量来减轻Reducer拉取数据的网络传输。  
④ : 最后将所有的溢出文件合并为一个文件，合并的过程中按照分区按照key进行排序（归并排序）, 如果溢出文件超过一定的数量（可配置)， 会在合并的前还会执行Combine操作（如果设置了Combiner）。  
⑤ : 当Map端有任务完成后，Reducer端就会启动对应的fetch & copy线程去从Map端复制数据。  
⑥ : 当Copy过来的数据内存中放不下后，会往磁盘写，写之前会先进行merge和sort操作（归并排序），combiner操作，最终会合并得到一份Reduce的输入数据。    
⑦ : 当输入数据准备好后，进行Reduce操作。  
⑧ : 输出数据到指定的位置。  

## Map Reduce的编程
Map 输入是<Key, Value>, 输出是一个或者多个<Key, Value>Reduce的输入是<Key, Iteratorable<Value>> 输出是<Key, Value>。总结来说:
> input<k1, v1>-->Map--><k2,v2>-->combine<k2,v2>-->Reduce--><k3, v3>(output)   

* 实现一个Map接口类
* 实现一个Reducer的接口类  

以Wordcount为例:  
**实现Map接口类**
```java
public static class TokenizerMapper extends Mapper<Object, Text, Text, IntWritable> {
	private final statck IntWritable one = new IntWritable(1);
	private Text word = new Text();
	
	public void Map(Object key, Text value, Context context) throws IOException, InterruptedException {
		StringTokenizer itr = new StringTokenizer(value.toString());
		while (itr.hasMoreTokens()) {
			word.set(itr.nextToken());
			context.write(word, one);
		}
	}
}
```
**实现Reducer接口类**
```java
public static class IntSumReducer extends Reducer<Text, IntWritable, Text, IntWriterable> {
	private IntWritable result = new IntWritable();
	
	public void Reduce(Text key, Iterable<IntWritable> values, Context context) throws IOException, InterruptedException {
		int sum = 0;
		for (IntWritable val: values) {
			sum += val.get();
		}
		result.set(sum);
		context.write(key, result);
	}
}
```
**处理任务的Job**
```java
public static void main(String[] args)throws Exception {
	Configuration conf = new Configuration();
	Job job = Job.getInstance(conf, "word count");
	job.setJarByClass(WordCount.class);
	job.setMapperClass(TokenizerMapper.class);
	job.setCombinerClass(IntSumReducer.class);
	job.setReducerClass(IntSumReducer.class);
	job.setOutputKeyClass(Text.class);
	job.setOutputValueClass(IntWritable.class);
	FileInputFormat.addInputPath(job, new Path(args[0]));
	FileOutputFormat.addOutputPath(job, new Path(args[1]));
	System.exit(job.waitForCompletion(true)?0:1);
}
```

**用Spark来实现Wordcount**
```scala
object WordCount {
	def main(args: Array[String]): Unit = {
		val conf = new SparkConf().setAppName("WordCount").setMaster("local[*]"))
		val spark = SparkSession.builder().config(conf).getOrCreate()
		
		import spark.implicits._
		val count = spark.read.textFile("input.txt")
				.flatMap(_.split(" "))
				.Map(s => (s, 1))
				.rdd
				.ReduceByKey((a, b) => a + b)
				.collect()
	}
}
```


## Map Reduce存在什么样的问题？
* 操作符或者算子固定单一只有Map和Reduce两种，无法支持复杂的操作，比如表之间`Join`操作，需要开发人员自己写`Join`的逻辑实现:
    * Reduce端的Join，在Map的时候对两个表的两一个key的数据分别打上表的标签，放进同一个key里，然后在Reduce阶段按标签分成两组，再进行Join输出操作。
    * Map端Join, 适合一端表小的情况，将小表在Map端作为其中一个lookup输入，进行Join操作。
* 读入输入数据然后产生的中间输出结果必须输出到磁盘，无法在中间内存中处理(Spark可以选择cache在内存里，用空间换时间)，重复写磁盘，I/O开销比较大。
* Key需要是可比较，会用来排序。
* Shuffle和Reduce阶段强制要求对数据按照某key进行排序，某些场景(比如数据量不是特别大的时候，简单hash就够了)会有性能的损失。
* 不能在线聚合，不论是Map端的combine还是Reduce端的都需要等所有的数据都存放到内存或者磁盘后，再执行聚合操作，存储这些数据需要消耗大量的内存和磁盘空间。如果能够一边获取record一边聚合，那么就会大大的减少存储空间，减少延时。

> **总结**:个人觉得Map Reduce主要的问题在于函数太过于底层，对用户的使用和操作上来说不够灵活，另外强制约束了需要按key排序和输出到磁盘使得其有性能上损失。但是也并不是全部一无是处，其中
* 简单清晰的分治MapReduce(Map即为分，Reduce即为合)进行分布式计算的思想被多个框架借鉴，各个阶段读什么数据，进行什么操作，输出数据都是确定的。
* 内存优势:
    * 内存使用固定，基本开销就是Map端输出数据的Spill Buffer，Reduce端需要一个大的数据来存放复制过来的分区数据两部分。用户聚合函数这部分的内存消耗是不确定的。
    * 排序后的数据在进行聚合的时候可以用最大堆或者最小堆来做，省空间且比较快。
    * 按照Key进行排序并Spill到磁盘的功能，可以保证Shuffle在大规模的数据时仍然能够顺利运行，不会那么容易出现OOM之类的问题。 
    * 通过归并排序来减少碎文件的提升I/O性能的思想其实也在Spark的SortMergeShuffle里使用。

### Spark是如何来解决这些问题的？
#### 提供丰富的操作符和分层的DAG并发处理层
* 通过抽象数据结构RDD和[丰富的操作符][2]来提升用户的操作体验。
    * 除了Map, Reduce，还提供了filter(), join(), cogroup(), flatMap(), union(), distinct()等等用户常用的操作符， 可参考Reference里Spark的transformation文档。

* 会将用户的代码分为**逻辑处理层**和**物理执行层**。
    * 逻辑处理层将用户的代码（定义的各种操作符）解析成一个DAG（有向无环图）来定义数据及数据之间的流动/操作关系。其中结点是RDD数据，箭头是在RDD上的一些数据操作和数据之间的关系。
    * 物理执行层会根据数据之间的依赖关系将DAG整个流程图划分成多个小的执行阶段（stage），然后按照各个执行阶段执行并处理数据。

#### 更灵活的Shuffle框架
* **解决强制按Key排序的问题**  
    * Spark提供按PartitionId排序、按Key排序等多种方式来灵活应对不同操作的排序需求。
* **提供在线聚合的方式**
    * 采用hash-based的聚合，利用HashMap的在线聚合特性，在将record插入HashMap时自动完成聚合过程，这也是Spark为什么设计AppendOnlyMap等数据结构的原因。
* **通过将最终临时文件合并成一个文件，按PartitionId顺序存储来减少碎文件**
    * Shuffle产生的临时文件会按照`PartitionId`去排序，最终会按照PartiontionId的顺序将一个Map产生的所有文件合成一个文件，来减少碎文件。

## 引申文档
* [HDFS](/_posts/2021-01-11-hdfs-in-hadoop.md)
* [YARN](/_posts/2021-01-11-yarn-in-hadoop.md)

## Reference
* [Hadoop文档][1]
* [Spark文档Transformation][2]

<b><font color="red">本网站的文章除非特别声明，全部都是原创。原创文章版权归数据元素</font>(</b>[DataElement](https://www.dataelement.top)<b><font color="red">)所有，未经许可不得转载!</font></b>
**了解更多大数据相关分享，可关注微信公众号"`数据元素`"**
![数据元素微信公众号](/img/dataelement.gif)

[1]: https://hadoop.apache.org/docs/r3.3.0/hadoop-MapReduce-client/hadoop-MapReduce-client-core/MapReduceTutorial.html
[2]: https://spark.apache.org/docs/latest/rdd-programming-guide.html#transformations

