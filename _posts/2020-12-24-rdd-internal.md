---
layout: post
title: "RDD Internal"
subtitle: "Deep Dive and Notes"
date: 2020-12-24
author: "Hanke"
header-style: "text"
tags: [Spark, Scala]
---

### RDD Five Main Properties
* A list of partitions
* A function for computing each split
* A list of dependencies on other RDDs
* Optionally, a partitioner for key-value RDDs (e.g. to say that the RDD is hash-partitioned)
    *  My own notes: the partitioner will be used for further shuffle check conditon
* Optionally, a list of preferred location to compute each split on (e.g. block partitions for a HDFS file)
    * The preferred location if it has, will be used for later data locality
