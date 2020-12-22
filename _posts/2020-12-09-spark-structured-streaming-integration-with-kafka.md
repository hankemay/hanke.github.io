---
layout: post
title: "Spark Structured Streaming Integration With Kafka"
subtitle: "Practice track for the kafka integration"
date: 2020-12-09
author: "Hanke"
header-style: "text"
tags: [Spark, Spark Structured Streaming]
---

### Error Messages Met
```bash
20/12/09 15:34:53 ERROR Executor: Exception in task 7.0 in stage 0.0 (TID 7)
java.lang.IllegalStateException: Cannot fetch offset 356071232 (GroupId: spark-kafka-source-5f4425dc-21b0-41ce-a938-ce8fa9e6bfee--1654482722-executor, TopicPartition: pair_prod_sample-45). 
Some data may have been lost because they are not available in Kafka any more; either the
 data was aged out by Kafka or the topic may have been deleted before all the data in the
 topic was processed. If you don't want your streaming query to fail on such cases, set the
 source option "failOnDataLoss" to "false".
```

```bash
org.apache.hadoop.fs.FileAlreadyExistsException: Rename destination file:/Users/hmxiao/workspace/kafka/kafka_2.13-2.6.0/output/.metadata.crc already exists.
```
Above error is caused by restarting the stream job later, and the offset is far away from current kafka beginning offset.


#### Kafka Consumer Setting
```bash
WARN InternalKafkaConsumerPool: Pool exceeds its soft max size, cleaning up idle objects...
```
**Solution**
`spark.kafka.consumer.cache.capacity` default is 64 in [Spark](https://spark.apache.org/docs/latest/structured-streaming-kafka-integration.html#consumer-caching).  

In the demo poc case, we have the 128 partitions in topic, better to set a larger cache capacity for it.
