---
layout: post
title: "Multiple Threads"
subtitle: "Notes of the thread usage in Spark"
date: 2020-12-17
author: "Hanke"
header-style: "text"
tags: [Java, Thread, Concurrent]
---

### Daemon

### countDownLatch
`countDownLatch` 会使一个线程等待其他线程各自执行完毕后再执行
* 通过一个计数器来实现，计数器的初始值是线程的数量。每当一个线程执行完毕后，计数器的值就-1， 当计数器的值为0时，表示所有线程都执行完毕，然后在闭锁上等待的线程就可以恢复工作了.  
Example:
```scala
private val startLatch = new CountDownLatch(1)

  /**
   * Starts the execution. This returns only after the thread has started and [[QueryStartedEvent]]
   * has been posted to all the listeners.
   */
  def start(): Unit = {
    logInfo(s"Starting $prettyIdString. Use $resolvedCheckpointRoot to store the query checkpoint.")
    queryExecutionThread.setDaemon(true)
    queryExecutionThread.start()
    startLatch.await()  // Wait until thread started and QueryStart event has been posted
  }


  val queryExecutionThread: QueryExecutionThread =
    new QueryExecutionThread(s"stream execution thread for $prettyIdString") {
      override def run(): Unit = {
        // To fix call site like "run at <unknown>:0", we bridge the call site from the caller
        // thread to this micro batch thread
        sparkSession.sparkContext.setCallSite(callSite)
        runStream()
      }
    }
```
Inside the queryExecutionThread, it will reduce the startLatch number in `runStream()`  
```scala
  private def runStream(): Unit = {
    try {
      sparkSession.sparkContext.setJobGroup(runId.toString, getBatchDescriptionString,
        interruptOnCancel = true)
      sparkSession.sparkContext.setLocalProperty(StreamExecution.QUERY_ID_KEY, id.toString)
      if (sparkSession.sessionState.conf.streamingMetricsEnabled) {
        sparkSession.sparkContext.env.metricsSystem.registerSource(streamMetrics)
      }

      // `postEvent` does not throw non fatal exception.
      val startTimestamp = triggerClock.getTimeMillis()
      postEvent(new QueryStartedEvent(id, runId, name, formatTimestamp(startTimestamp)))

      // Unblock starting thread
      startLatch.countDown()
...........
}
```

### ReentratLock
和Synchonized的作用类似, 是一个独占锁, 但也有一些不同的地方.  

**不同**:
* Synchonized自动加锁释放锁, 不够灵活，而ReentratLock需要手动加锁释放锁, 相对灵活一些
* Synchonized可重入, 且不必担心最后是否释放锁，而ReentratLock可以重入，但加锁和解锁需要手动进行, 且次数需要一样, 否则其他线程拿不到锁
* Synchonized不可以响应中断, 一个线程如果拿不到锁会一直等着; 而ReentratLock可以响应中断, eg. `thread.interrupt()` 和 `lock.lockInterruptibly()`
* Synchonized没有公平锁，而ReentratLock有公平锁, 在锁上等待时间最长的线程将获得锁的使用权.

```scala
  /**
   * A lock used to wait/notify when batches complete. Use a fair lock to avoid thread starvation.
   */
  protected val awaitProgressLock = new ReentrantLock(true)
  protected val awaitProgressLockCondition = awaitProgressLock.newCondition()
```

How to use it?
```scala
....
      } finally {
        awaitProgressLock.lock()
        try {
          // Wake up any threads that are waiting for the stream to progress.
          awaitProgressLockCondition.signalAll()
        } finally {
          awaitProgressLock.unlock()
        }
        terminationLatch.countDown()
      }
```
