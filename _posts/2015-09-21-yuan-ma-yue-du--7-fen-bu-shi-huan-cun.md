---
layout: post
title: "源码阅读 （7）分布式缓存"
description: ""
category: "spark"
tags: [spark]
---
{% include JB/setup %}
在Spark中要缓存中间结果时我们会调用RDD的cache方法。那我们来看一下RDD中的cache方法。    

{% highlight scala linenos %}
/** Persist this RDD with the default storage level (`MEMORY_ONLY`). */
def cache(): this.type = persist()
{% endhighlight %}

那我们来看一下persist方法。    

{% highlight scala linenos %}

/** Persist this RDD with the default storage level (`MEMORY_ONLY`). */
def persist(): this.type = persist(StorageLevel.MEMORY_ONLY)

/**
 * Set this RDD's storage level to persist its values across operations after the first time
 * it is computed. This can only be used to assign a new storage level if the RDD does not
 * have a storage level set yet. Local checkpointing is an exception.
 */
def persist(newLevel: StorageLevel): this.type = {
  if (isLocallyCheckpointed) {
    // This means the user previously called localCheckpoint(), which should have already
    // marked this RDD for persisting. Here we should override the old storage level with
    // one that is explicitly requested by the user (after adapting it to use disk).
    persist(LocalRDDCheckpointData.transformStorageLevel(newLevel), allowOverride = true)
  } else {
    persist(newLevel, allowOverride = false)
  }
}


/**
 * Mark this RDD for persisting using the specified level.
 *
 * @param newLevel the target storage level
 * @param allowOverride whether to override any existing level with the new one
 */
private def persist(newLevel: StorageLevel, allowOverride: Boolean): this.type = {
  // TODO: Handle changes of StorageLevel
  if (storageLevel != StorageLevel.NONE && newLevel != storageLevel && !allowOverride) {
    throw new UnsupportedOperationException(
      "Cannot change storage level of an RDD after it was already assigned a level")
  }
  // If this is the first time this RDD is marked for persisting, register it
  // with the SparkContext for cleanups and accounting. Do this only once.
  // 如果这是第一次标记这个RDD是持久化的，那么为了清理与核算，
  // 在SparkContext中为它注册。只执行一次。
  if (storageLevel == StorageLevel.NONE) {
    sc.cleaner.foreach(_.registerRDDForCleanup(this))
    //调用SparkContext去缓存这个RDD
    sc.persistRDD(this)
  }
  storageLevel = newLevel
  this
}
{% endhighlight %}

SparkContext中的persistRDD代码：
{% highlight scala linenos %}
/**
 * Register an RDD to be persisted in memory and/or disk storage
 */
private[spark] def persistRDD(rdd: RDD[_]) {
  _executorAllocationManager.foreach { _ =>
    logWarning(
      s"Dynamic allocation currently does not support cached RDDs. Cached data for RDD " +
      s"${rdd.id} will be lost when executors are removed.")
  }
  persistentRdds(rdd.id) = rdd
}
{% endhighlight %}

其中persistentRdds是一个map

{% highlight scala linenos %}
private[spark] val persistentRdds = new TimeStampedWeakValueHashMap[Int, RDD[_]]
{% endhighlight %}

现在只是声明这个RDD需要缓存，具体需要到RDD执行的时候才能被缓存。入口是Task的runTask方法。

以ResultTask为例
{% highlight scala linenos %}
override def runTask(context: TaskContext): U = {
  // Deserialize the RDD and the func using the broadcast variables.
  val deserializeStartTime = System.currentTimeMillis()
  val ser = SparkEnv.get.closureSerializer.newInstance()
  val (rdd, func) = ser.deserialize[(RDD[T], (TaskContext, Iterator[T]) => U)](
    ByteBuffer.wrap(taskBinary.value), Thread.currentThread.getContextClassLoader)
  _executorDeserializeTime = System.currentTimeMillis() - deserializeStartTime

  metrics = Some(context.taskMetrics)
  func(context, rdd.iterator(partition, context))
}
{% endhighlight %}

它调用的RDD的iterator方法

{% highlight scala linenos %}
/**
 * Internal method to this RDD; will read from cache if applicable, or otherwise compute it.
 * This should ''not'' be called by users directly, but is available for implementors of custom
 * subclasses of RDD.
 */
final def iterator(split: Partition, context: TaskContext): Iterator[T] = {
  if (storageLevel != StorageLevel.NONE) {
    SparkEnv.get.cacheManager.getOrCompute(this, split, context, storageLevel)
  } else {
    computeOrReadCheckpoint(split, context)
  }
}
{% endhighlight %}

如果设置的storageLevel那么就从CacheManager中取数据。那么我们就来看看CacheManager中的getOrCompute方法：

{% highlight scala linenos %}
/** Gets or computes an RDD partition. Used by RDD.iterator() when an RDD is cached. */
def getOrCompute[T](
    rdd: RDD[T],
    partition: Partition,
    context: TaskContext,
    storageLevel: StorageLevel): Iterator[T] = {

  val key = RDDBlockId(rdd.id, partition.index)
  logDebug(s"Looking for partition $key")
  blockManager.get(key) match {
    // 已经缓存好了，更新度量信息后直接返回
    case Some(blockResult) =>
      // Partition is already materialized, so just return its values
      val existingMetrics = context.taskMetrics
        .getInputMetricsForReadMethod(blockResult.readMethod)
      existingMetrics.incBytesRead(blockResult.bytes)

      val iter = blockResult.data.asInstanceOf[Iterator[T]]
      new InterruptibleIterator[T](context, iter) {
        override def next(): T = {
          existingMetrics.incRecordsRead(1)
          delegate.next()
        }
      }
    case None =>
      // Acquire a lock for loading this partition
      // If another thread already holds the lock, wait for it to finish return its results
      // 获取加载当前分区的锁，如果已经有其他线程获取到了锁，那么等它结束后获取它的结果
      val storedValues = acquireLockForPartition[T](key)
      if (storedValues.isDefined) {
        return new InterruptibleIterator[T](context, storedValues.get)
      }
    
      // Otherwise, we have to load the partition ourselves
      // 不然，我们就得自己加载这个分区。
      try {
        logInfo(s"Partition $key not found, computing it")
        val computedValues = rdd.computeOrReadCheckpoint(partition, context)

        // If the task is running locally, do not persist the result
        // 如果任务是在本地运行的，那么就不需要持久化结果
        if (context.isRunningLocally) {
          return computedValues
        }

        // Otherwise, cache the values and keep track of any updates in block statuses
        // 不然，缓存结果，并跟踪块的任何信息变更。
        val updatedBlocks = new ArrayBuffer[(BlockId, BlockStatus)]
        val cachedValues = putInBlockManager(key, computedValues, storageLevel, updatedBlocks)
        val metrics = context.taskMetrics
        val lastUpdatedBlocks = metrics.updatedBlocks.getOrElse(Seq[(BlockId, BlockStatus)]())
        metrics.updatedBlocks = Some(lastUpdatedBlocks ++ updatedBlocks.toSeq)
        new InterruptibleIterator(context, cachedValues)

      } finally {
        loading.synchronized {
          loading.remove(key)
          loading.notifyAll()
        }
      }
  }
}
{% endhighlight %}

这里使用putInBlockManager把计算结果缓存如blockManager。
