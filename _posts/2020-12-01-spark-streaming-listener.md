---
layout: post
title: "Spark StreamingQueryListener"
subtitle: "Customized Spark Streaming Query Listener Implementation"
author: "Hanke"
header-style: text
tags: [Spark, Scala, Spark Structured Streaming]
---

This blog will explain the spark streaming query listener interface, and how to implement one customized listener for steaming cases. What's more, how to call it in the streaming job.

### Listener Interface
```scala
org.apache.spark.sql.streaming.StreamingQueryListener

abstract class StreamingQueryListener {

  import StreamingQueryListener._

  /**
   * Called when a query is started.
   * @note This is called synchronously with
   *       [[org.apache.spark.sql.streaming.DataStreamWriter `DataStreamWriter.start()`]],
   *       that is, `onQueryStart` will be called on all listeners before
   *       `DataStreamWriter.start()` returns the corresponding [[StreamingQuery]]. Please
   *       don't block this method as it will block your query.
   * @since 2.0.0
   */
  def onQueryStarted(event: QueryStartedEvent): Unit

  /**
   * Called when there is some status update (ingestion rate updated, etc.)
   *
   * @note This method is asynchronous. The status in [[StreamingQuery]] will always be
   *       latest no matter when this method is called. Therefore, the status of [[StreamingQuery]]
   *       may be changed before/when you process the event. E.g., you may find [[StreamingQuery]]
   *       is terminated when you are processing `QueryProgressEvent`.
   * @since 2.0.0
   */
  def onQueryProgress(event: QueryProgressEvent): Unit

  /**
   * Called when a query is stopped, with or without error.
   * @since 2.0.0
   */
  def onQueryTerminated(event: QueryTerminatedEvent): Unit
}
```

### StreamQueryListener Implementation

```scala
class StreamQueryListener(val query: StreamingQuery, val maxEmptyTicks: Int = 3) extends StreamingQueryListener {

  private val queryId           = query.id
  private var currentEmptyCount = 0
  private var totalCount: Long  = 0

  override def onQueryStarted(event: StreamingQueryListener.QueryStartedEvent): Unit = {
    if (event.id == queryId) {
      !s"Query started. (id = $queryId)"
    }
  }

  override def onQueryProgress(event: StreamingQueryListener.QueryProgressEvent): Unit = {
    if (event.progress.id == queryId) {
      !s"Query porgress. (id = $queryId)\n\tNumber of input rows = ${event.progress.numInputRows}, currentEmptyCount = $currentEmptyCount (total count = ${totalCount + event.progress.numInputRows})"
      event.progress.numInputRows match {
        case 0 =>
          currentEmptyCount += 1
          checkCounterLimit()
        case x =>
          currentEmptyCount = 0
          totalCount += x
      }
    }
  }
  private def checkCounterLimit(): Unit = {
    if (currentEmptyCount >= maxEmptyTicks) {
      !s"Query will be STOPPED! (id = $queryId)"
      query.stop()
    }
  }
  override def onQueryTerminated(event: StreamingQueryListener.QueryTerminatedEvent): Unit = {
    if (event.id == queryId) {
      !s"Query terminated. (id = $queryId)\n\tTotal rows processed= $totalCount"
    }
  }
}
```

```scala
spark.streams.addListener(new StreamQueryListener(fileStream, maxRetrives))
```

-----

Want to see something else added? <a href="https://dataelement.top">Open an issue.</a>
