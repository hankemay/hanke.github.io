---
layout: post
title: Spark Join
subtitle: Details of the spark join
header-style: text
author: Hanke
tags: [Spark, Spark SQL]
---
This document will explain the DB join and spark join.

[TOC]
### DB Join
#### SQL Join Types
##### Inner or Outer Joins
##### EQUI Join
##### NON EQUI Join (Theta Join)

#### Join Implementation
##### Nested Loop Join
##### Hash Join
##### Sort Merged Join


#### Optimizer – Select Join Strategy
##### Cost Model

### Spark Join
#### Join Syntax
##### Supported Join Types
#### Join Methods
##### Shuffle Hash Join
##### Broadcast Hash Join (Map Join)
###### Spark SQL Config
###### Http broadcast VS TorrentBroadcast (Based on BitTorrent)
###### Driver Broadcast Join
###### Executor Broadcast Join
##### Shuffle Sort Merge Join (Default)
##### Cartesian Product Join (Shuffle Replicate Nested Loop Join)
##### Broadcast Nested Loop Join
#### Scenarios
#### Optimizer
##### CBO
###### Statistics Collection
###### Spark SQL Config
###### Operator Cost Formula
###### Mainly Operator Estimation
###### Benefits
##### Join Hints
##### Select Join Strategy
#### Spark 3.0 AQE
##### Dynamically coalesce shuffle partitions
##### Dynamically switch join strategies
##### Dynamically optimize skew joins 
### Suggestions
### Reference


-----

Want to see something else added? <a href="https://hanke.github.io">Open an issue.</a>
