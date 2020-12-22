---
layout: post
title: "Notes of Scala Usage in Spark"
subtitle: "What I Saw"
date: 2020-12-18
author: "Hanke"
header-style: "text"
tags: [Scala, Java]
---

### Sealed Trait
**作用**: 
* 为了防止case class match的时候遗漏某些cases, 如果有遗漏会报错  
**要求**:  
* 定义的Trait和继承类的需要在同一个文件里  


Example:
```scala
sealed trait MultipleWatermarkPolicy {
  def chooseGlobalWatermark(operatorWatermarks: Seq[Long]): Long
}

case object MinWatermark extends MultipleWatermarkPolicy {
  def chooseGlobalWatermark(operatorWatermarks: Seq[Long]): Long = {
    assert(operatorWatermarks.nonEmpty)
    operatorWatermarks.min
  }
}

case object MaxWatermark extends MultipleWatermarkPolicy {
  def chooseGlobalWatermark(operatorWatermarks: Seq[Long]): Long = {
    assert(operatorWatermarks.nonEmpty)
    operatorWatermarks.max
  }
}

```


### 变量赋值里的下划线
> scala里的变量声明的时候需要给初始值，可以使用下划线对变量进行初始化  

Example:
```
private var watermarkTracker: WatermarkTracker = _
```


### Java调用Scala里的Module$
> 经常在spark源码里会读到Module$这样的代码，比较疑惑是做什么的.  

```java
/**
 * OutputMode describes what data will be written to a streaming sink when there is
 * new data available in a streaming DataFrame/Dataset.
 *
 * @since 2.0.0
 */
@Evolving
public class OutputMode {

  /**
   * OutputMode in which only the new rows in the streaming DataFrame/Dataset will be
   * written to the sink. This output mode can be only be used in queries that do not
   * contain any aggregation.
   *
   * @since 2.0.0
   */
  public static OutputMode Append() {
    return InternalOutputModes.Append$.MODULE$;
  }

  /**
   * OutputMode in which all the rows in the streaming DataFrame/Dataset will be written
   * to the sink every time there are some updates. This output mode can only be used in queries
   * that contain aggregations.
   *
   * @since 2.0.0
   */
  public static OutputMode Complete() {
    return InternalOutputModes.Complete$.MODULE$;
  }

  /**
   * OutputMode in which only the rows that were updated in the streaming DataFrame/Dataset will
   * be written to the sink every time there are some updates. If the query doesn't contain
   * aggregations, it will be equivalent to `Append` mode.
   *
   * @since 2.1.1
   */
  public static OutputMode Update() {
    return InternalOutputModes.Update$.MODULE$;
  }
}
```
[参考文章](https://blog.csdn.net/zhangjg_blog/article/details/23376465),搜到这篇文章写的还是比较全.

由此可见，实际上面的示例代码返回的自身的一个单例对象. 只是这个地方没太明白的是，为什么不用scala里的enum来表示，而要用这样的类继承的方式呢？

### Scala Apply
Scala里用apply来去构造对象，很多时候也会变成工厂方法.
```scala
object JoinType {
  def apply(typ: String): JoinType = typ.toLowerCase(Locale.ROOT).replace("_", "") match {
    case "inner" => Inner
    case "outer" | "full" | "fullouter" => FullOuter
    case "leftouter" | "left" => LeftOuter
    case "rightouter" | "right" => RightOuter
    case "leftsemi" | "semi" => LeftSemi
    case "leftanti" | "anti" => LeftAnti
    case "cross" => Cross
    case _ =>
      val supported = Seq(
        "inner",
        "outer", "full", "fullouter", "full_outer",
        "leftouter", "left", "left_outer",
        "rightouter", "right", "right_outer",
        "leftsemi", "left_semi", "semi",
        "leftanti", "left_anti", "anti",
        "cross")

      throw new IllegalArgumentException(s"Unsupported join type '$typ'. " +
        "Supported join types include: " + supported.mkString("'", "', '", "'") + ".")
  }
}

sealed abstract class JoinType {
  def sql: String
}

/**
 * The explicitCartesian flag indicates if the inner join was constructed with a CROSS join
 * indicating a cartesian product has been explicitly requested.
 */
sealed abstract class InnerLike extends JoinType {
  def explicitCartesian: Boolean
}

case object Inner extends InnerLike {
  override def explicitCartesian: Boolean = false
  override def sql: String = "INNER"
}

case object Cross extends InnerLike {
  override def explicitCartesian: Boolean = true
  override def sql: String = "CROSS"
}
```

### Scala Unapply
scala里的`apply`是用来构造对象的(不用new的方式), `unapply`是用来提取取象的值(和apply是反向的). `unapply`主要用在match的时候的，case Object(a, b, c) 这样，提取出object里面的a, b, c的值.


### Scala Self-Type
> spark 源码里关于scala self-type  

```scala
abstract class TreeNode[BaseType <: TreeNode[BaseType]] extends Product {
// scalastyle:on
  self: BaseType =>

  val origin: Origin = CurrentOrigin.get
```
其中 `self: BaseType =>` 要求TreeNode本身也是一个BaseType  

**要求**
* 如果一个类或者trait指明了self-type, 那么要求它的子类型或者是对象也必须是相应的类型。  

**作用**
* 当一个特质仅用于混入一个或者几个特质时，可以指定自身类型来进行限制
* 简单的依赖注入？可以用来表达组件和组件之间的依赖关系


Self-type [参考文章](https://www.colabug.com/2016/0317/666838/)





