---
layout: post
title: "Companion Object in Scala"
subtitle: "why we need the the companion object?"
date: 2020-12-16
author: "Hanke"
header-style: "text"
tags: [Scala]
---

> 阅读spark源码的时候，看到很多地方都是对象和伴生对象在一起，好奇为什么需要伴生对象?

### 原因
scala没有`static`关键字，但是对于类很多时候需要静态变量或者方法，所以伴生对象很多时间扮演这个补充角色
* 对象负责定义实例成员变量和方法，而伴生对象负责定义静态变量和方法
* 伴生对象里的`apply`方法，也可以间接扮演工厂方法的角色，这样new一个对象的时候不需要用new，直接伴生对象()即可

### 好处
* 补充`static`场景功能, class和object可以互相访问对方的私有方法和变量
* 资源共享，节省存储空间(JVM为静态属性单独开辟一块空间即可)
* 方便new一个对象

### 要求
* class和object要同名且在一个文件中


### 参考源码  
```scala
/**
 * Information about progress made for a sink in the execution of a [[StreamingQuery]]
 * during a trigger. See [[StreamingQueryProgress]] for more information.
 *
 * @param description Description of the source corresponding to this status.
 * @param numOutputRows Number of rows written to the sink or -1 for Continuous Mode (temporarily)
 * or Sink V1 (until decommissioned).
 * @since 2.1.0
 */
@Evolving
class SinkProgress protected[sql](
    val description: String,
    val numOutputRows: Long) extends Serializable {

  /** SinkProgress without custom metrics. */
  protected[sql] def this(description: String) {
    this(description, DEFAULT_NUM_OUTPUT_ROWS)
  }

  /** The compact JSON representation of this progress. */
  def json: String = compact(render(jsonValue))

  /** The pretty (i.e. indented) JSON representation of this progress. */
  def prettyJson: String = pretty(render(jsonValue))

  override def toString: String = prettyJson

  private[sql] def jsonValue: JValue = {
    ("description" -> JString(description)) ~
      ("numOutputRows" -> JInt(numOutputRows))
  }
}

private[sql] object SinkProgress {
  val DEFAULT_NUM_OUTPUT_ROWS: Long = -1L

  def apply(description: String, numOutputRows: Option[Long]): SinkProgress =
    new SinkProgress(description, numOutputRows.getOrElse(DEFAULT_NUM_OUTPUT_ROWS))
}
```
