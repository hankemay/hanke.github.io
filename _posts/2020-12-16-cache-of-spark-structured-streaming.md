---
layout: post
title: "Cache of Spark Structured Streaming"
subtitle: "Support or Not?"
date: 2020-12-16
author: "Hanke"
header-style: "text"
tags: [Spark, Spark Structured Streaming]
---

### Update the Static Data Periodic
Try the `StreamingQueryListerner`, generate the temp table, persist/unpersist?
Example Code  

```scala
// Read metadata from MySQL, the metadata is to be joined with Kafka real time message
// 从 MySQL 读的数据会缓慢变化，比如隔了几天新增一些数据
val df_meta = spark.read
  .format("jdbc")
  .option("url", mysql_url)
  .option("dbtable", "v_entity_ap_rel")
  .load()

val df_cell = spark.read
  .format("jdbc")
  .option("url", mysql_url)
  .option("dbtable", "tmac_org_cat")
  .load()
  .filter($"Category" === 1)
  .select("mac3", "Category", "Brand")

df_meta.cache()
df_cell.cache()

// Read Kafka and join with df_meta to get expect result
// Kafka 里面是 Mac 地址和采集的相关数据，每行 Kafka 的消息会根上面两个 cache
// 的 DataFrame join 到一起形成最终需要的结果，问题就是 struct streaming
// 一直在运行，而读 MySQL 是一次性的，除非重新启动 Spark
// 程序，如何能够隔一段时间 reload MySQL 里面的数据？
val df = spark.readStream.
  .format("kafka")
  .option("kafka.bootstrap.servers", "namenode:9092")
  .option("fetch.message.max.bytes", "50000000")
  .option("kafka.max.partition.fetch.bytes", "50000000")
  .option("subscribe", "rawdb.raw_data")
  .option("failOnDataLoss", true)
  .option("startingOffsets", "latest")
  .load()
  .select($"value".as[Array[Byte]])
  .map(avroDeserialize(_))
  .as[ApRawData]
  .selectExpr("apMacAddress APMAC", "rssi RRSSI", "sourceMacAddress CLIENTMAC", "time STIME", "updatingTime")
  .filter($"stime".lt(current_timestamp()))
  .filter($"stime".gt(from_unixtime(unix_timestamp(current_timestamp()).minus(5 * 60))))
  .as("d")
  .join(df_cell.as("c"), substring($"d.CLIENTMAC", 1, 6) === $"c.mac3")
  .as("a")
  .join(df_meta.as("b"), $"a.apmac" === $"b.apmac")

val query = df
  .selectExpr(
    "ENTITYID",
    "CLIENTMAC",
    "STIME",
    "CASE WHEN a.rrssi >= b.rssi THEN '1' WHEN a.rrssi < b.nearbyrssi THEN '3' ELSE '2' END FLAG",
    "substring(stime, 1, 10) DATE",
    "substring(stime, 12, 2) HOUR")
  .repartition(col("DATE"), col("HOUR"))
  .writeStream
  .format("csv")
  .option("header",false)
  .partitionBy("DATE", "HOUR")
  .option("checkpointLocation", "/user/root/t_cf_table_chpt")
  .trigger(ProcessingTime("5 minutes"))
  .start("T_CF_TABLE")
  .awaitTermination()
```

### Use the ForeachBatch
```shell
org.apache.spark.sql.streaming.StreamingQueryException: Queries with streaming sources must be executed with writeStream.start();;
```

### Reference
* [Jira Ticket](https://issues.apache.org/jira/browse/SPARK-20927)
* [Update the static data](https://www.zhihu.com/question/264637955)
