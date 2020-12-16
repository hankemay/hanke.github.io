---
layout: post
title: "Notes of Spark Structured Streaming"
subtitle: "Some keynotes of structured streaming"
date: 2020-12-16
author: "Hanke"
header-style: "text"
tags: [Spark, Spark Structured Streaming]
---


### Deduplicate
Removing duplicates **bounded** by a watermark

### Custom Stateful Processing
Use `mapGroupsWithState` and `flatMapGroupsWithState` to customized your stateful processing

### Reference
* [Arbitary stateful processing](https://databricks.com/blog/2017/10/17/arbitrary-stateful-processing-in-apache-sparks-structured-streaming.html)
