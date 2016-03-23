---
layout: post
title: "源码阅读 （5）Stage生成"
description: ""
category: ""
tags: [spark]
---
{% include JB/setup %}

###Stage生成
  Stage的调度是由DAGScheduler承担的。由RDD的DAG切分出了Stage的DAG。Stage的DAG通过最后执行的Stage为根进行广度优先遍历，遍历到最开始的Stage再执行。如果提交的Stage有仍未完成的父Stage，那么Stage需要等待其完成才能执行。    
  DAGScheduler中还维护了几个重要的Key-Value集合结构，用来记录Stage的状态，这样能够避免过早执行或重复提交。
  
> waitingStages  记录仍有未执行父Stage的Stage，防止过早执行    
> runningStages  保存正在执行的Stage，防止重复执行    
> failedStages   保存执行失败Stage，需要重复执行    

~~~ scala
// Stages we need to run whose parents aren't done
private[scheduler] val waitingStages = new HashSet[Stage]

// Stages we are running right now
private[scheduler] val runningStages = new HashSet[Stage]

// Stages that must be resubmitted due to fetch failures
private[scheduler] val failedStages = new HashSet[Stage]
~~~

###依赖关系
调度会计算RDD之间的依赖关系，将拥有持续窄依赖的RDD归并到同一个Stage中，而宽依赖则作为划分不同Stage的标准。

###Stage分类

####ShuffleMapStage
    非最终Stage，后面还有其他Stage。他的输出需要shuffle之后才能作为后续输入。
    以shuffle为输出边界。其数据可以从外部获取，也可以是另一个ShuffleMapStage的输出。
    最后的Task是ShuffleMapTask。
    一个Job中可能有也可能没有这种Stage

####ResultStage
     最终Stage，没有输出，直接产生结果或存储。
     直接输出结果，其数据可以从外部获取，也可以是ShuffleMapStage的输出。
     最后的Task是ResultTask
     一个Job中必须有这种Stage
    
###Stage划分
RDD转换本身存在ShuffleDependency，像ShuffleRDD，CoGroupRDD，SubtractedRDD会返回ShuffleDependency。
如果RDD中存在ShuffleDependency，就会创建一个新的Stage。

Stage划分完毕就明确了一下东西：     

> 产生的Stage需要从多少个Partition中读取数据    
> 产生的Stage会生产多少个Partition    
> 产生的Stage是否属于ShuffleMap类型    

Partition用来决定需要产生多少不同的Task，是否为ShuffleMap类型用来确定生成的Task类型。
Spark有两种Task:    
 1. ShuffleMapTask    
 2. ResultTask

###Stage类
因为在一个Stage中所有RDD都是map，parition不会有任何改变，所以Stage类中只有一个（而不是一系列）RDD参数。（只在data上依次执行不同的map function）

~~~ scala
private[spark] abstract class Stage(
    val id: Int,      //Stage序号，数值越大，优先级越高
    val rdd: RDD[_],  //归属于本Stage的最后一个rdd
    val numTasks: Int,        //创建的Task数目，等于父RDD的输出Partition数目
    val parents: List[Stage],  //父Stage列表
    val firstJobId: Int,        //作业ID
    val callSite: CallSite)
  extends Logging {
~~~

###Job处理    
1. 分割Job为Stage
2. 封装Stage成TaskSet
3. 提交给TaskScheduler

####DAGScheduler的handleJobSubmited、submitStage函数
这两个函数主要负责依赖分析，对其处理逻辑做进一步的分析。
handleJobSubmited的主要工作是生产Stage，并根据finalStage来产生ActiveJob。

~~~ scala
private[scheduler] def handleJobSubmitted(jobId: Int,
    finalRDD: RDD[_],
    func: (TaskContext, Iterator[_]) => _,
    partitions: Array[Int],
    callSite: CallSite,
    listener: JobListener,
    properties: Properties) {
  var finalStage: ResultStage = null
  try {
    // New stage creation may throw an exception if, for example, jobs are run on a
    // HadoopRDD whose underlying HDFS files have been deleted.
    finalStage = newResultStage(finalRDD, partitions.length, jobId, callSite)
  } catch {
    //错误处理，告诉监听器作业失败，并返回
    case e: Exception =>
      logWarning("Creating new stage failed due to exception - job: " + jobId, e)
      listener.jobFailed(e)
      return
  }
  if (finalStage != null) {
    //创建ActiveJob
    val job = new ActiveJob(jobId, finalStage, func, partitions, callSite, listener, properties)
    clearCacheLocs()
    logInfo("Got job %s (%s) with %d output partitions".format(
      job.jobId, callSite.shortForm, partitions.length))
    logInfo("Final stage: " + finalStage + "(" + finalStage.name + ")")
    logInfo("Parents of final stage: " + finalStage.parents)
    logInfo("Missing parents: " + getMissingParentStages(finalStage))
    val jobSubmissionTime = clock.getTimeMillis()
    jobIdToActiveJob(jobId) = job
    activeJobs += job
    finalStage.resultOfJob = Some(job)
    val stageIds = jobIdToStageIds(jobId).toArray
    val stageInfos = stageIds.flatMap(id => stageIdToStage.get(id).map(_.latestInfo))
    listenerBus.post(
      SparkListenerJobStart(job.jobId, jobSubmissionTime, stageInfos, properties))
    //提交stage
    submitStage(finalStage)
  }
  //提交stage
  submitWaitingStages()
}
~~~

####newResultStage函数
~~~ scala
/**
 * Create a ResultStage associated with the provided jobId.
 */
private def newResultStage(
    rdd: RDD[_],
    numTasks: Int,
    jobId: Int,
    callSite: CallSite): ResultStage = {
  val (parentStages: List[Stage], id: Int) = getParentStagesAndId(rdd, jobId)
  val stage: ResultStage = new ResultStage(id, rdd, numTasks, parentStages, jobId, callSite)

  stageIdToStage(id) = stage
  updateJobIdStageIdMaps(jobId, stage)
  stage
}
~~~
创建Stage需要知道该Stage需要从多少Partition读入数据，这个数字直接影响需要创建多少个Task。也就是说创建Stage的时候就已经清楚该Stage需要从多少个不同的Partition读入数据，并写到多少个Partition中，输入输出的个数都已经明确。
~~~ scala
/**
 * Helper function to eliminate some code re-use when creating new stages.
 */
private def getParentStagesAndId(rdd: RDD[_], firstJobId: Int): (List[Stage], Int) = {
  val parentStages = getParentStages(rdd, firstJobId)
  val id = nextStageId.getAndIncrement()
  (parentStages, id)
}
~~~
####getParentStages函数
~~~ scala
/**
 * Get or create the list of parent stages for a given RDD.  The new Stages will be created with
 * the provided firstJobId.
 */
private def getParentStages(rdd: RDD[_], firstJobId: Int): List[Stage] = {
  val parents = new HashSet[Stage]
  val visited = new HashSet[RDD[_]]
  // We are manually maintaining a stack here to prevent StackOverflowError
  // caused by recursively visiting
  val waitingForVisit = new Stack[RDD[_]]
  def visit(r: RDD[_]) {
    if (!visited(r)) {
      visited += r
      // Kind of ugly: need to register RDDs with the cache here since
      // we can't do it in its constructor because # of partitions is unknown
      for (dep <- r.dependencies) {
        dep match {
          case shufDep: ShuffleDependency[_, _, _] =>
            parents += getShuffleMapStage(shufDep, firstJobId)
          case _ =>
            waitingForVisit.push(dep.rdd)
        }
      }
    }
  }
  waitingForVisit.push(rdd)
  while (waitingForVisit.nonEmpty) {
    visit(waitingForVisit.pop())
  }
  parents.toList
}
~~~
通过不停的遍历他之前的RDD,如果碰到有依赖是ShuffleMapDependency类型的，就通过getShuffleMapStage方法计算出它的Stage来。

###ActiveJob类

用户所提交的job在得到DAGScheduler的调度后,会被
~~~ scala
val job = new ActiveJob(jobId, finalStage, func, partitions, callSite, listener, properties)
~~~
包装成ActiveJob,同时会启动JobWaiter阻塞监听job的完成状况。    
~~~ scala
private[spark] class ActiveJob(
    val jobId: Int,
    val finalStage: ResultStage,
    val func: (TaskContext, Iterator[_]) => _,
    val partitions: Array[Int],
    val callSite: CallSite,
    val listener: JobListener,
    val properties: Properties) {

  val numPartitions = partitions.length
  val finished = Array.fill[Boolean](numPartitions)(false)
  var numFinished = 0
}
~~~

依据job中RDD的dependency和dependency属性(NarrowDependency，ShufflerDependecy)，DAGScheduler会根据依赖关系的先后产生出不同的stage DAG(result stage, shuffle map stage)。
在每一个stage内部，根据stage产生出相应的task，包括ResultTask或是ShuffleMapTask，这些task会根据RDD中partition的数量和分布，产生出一组相应的task，并将其包装为TaskSet提交到TaskScheduler上去。

###submitStage函数    

submitStage函数中会根据依赖关系划分stage，通过递归调用从finalStage一直往前找它的父stage，直到stage没有父stage时就调用submitMissingTasks方法提交改stage。这样就完成了将job划分为一个或者多个stage。

submitStage处理流程：    

> 所依赖的Stage是否都已经完成，如果没有完成则先执行所依赖的Stage
> 如果所有的依赖已经完成，则提交自身所处的Stage
> 最后会在submitMissingTasks函数中将stage封装成TaskSet通过taskScheduler.submitTasks函数提交给TaskScheduler处理。

~~~ scala
  /** Submits stage, but first recursively submits any missing parents. */
  private def submitStage(stage: Stage) {
    val jobId = activeJobForStage(stage)
    if (jobId.isDefined) {
      logDebug("submitStage(" + stage + ")")
      if (!waitingStages(stage) && !runningStages(stage) && !failedStages(stage)) {
        val missing = getMissingParentStages(stage).sortBy(_.id)
        logDebug("missing: " + missing)
        if (missing.isEmpty) {
          logInfo("Submitting " + stage + " (" + stage.rdd + "), which has no missing parents")
          submitMissingTasks(stage, jobId.get)  // 如果没有parent stage需要执行, 则直接submit当前stage的task
        } else {
          for (parent <- missing) {
            submitStage(parent) // 提交父stage的task，这里是个递归，直到没有父stage才在上面的语句中提交task
          }
          waitingStages += stage // 暂时不能提交的stage，先添加到等待队列
        }
      }
    } else {
      abortStage(stage, "No active job for stage " + stage.id, None)
    }
  }
~~~
这个提交stage的过程是一个递归的过程，它是先要把父stage先提交，然后把自己添加到等待队列中，直到没有父stage之后，就提交该stage中的任务。等待队列在最后的submitWaitingStages方法中提交。

###getMissingParentStages函数
getMissingParentStages通过图的遍历，来找出所依赖的所有父Stage。

~~~ scala
  private def getMissingParentStages(stage: Stage): List[Stage] = {
    val missing = new HashSet[Stage]
    val visited = new HashSet[RDD[_]]
    // We are manually maintaining a stack here to prevent StackOverflowError
    // caused by recursively visiting
    val waitingForVisit = new Stack[RDD[_]]
    def visit(rdd: RDD[_]) {
      if (!visited(rdd)) {
        visited += rdd
        val rddHasUncachedPartitions = getCacheLocs(rdd).contains(Nil)
        if (rddHasUncachedPartitions) {
          for (dep <- rdd.dependencies) {
            dep match {
              case shufDep: ShuffleDependency[_, _, _] =>
                val mapStage = getShuffleMapStage(shufDep, stage.firstJobId)
                if (!mapStage.isAvailable) {
                  missing += mapStage
                }
              case narrowDep: NarrowDependency[_] =>
                waitingForVisit.push(narrowDep.rdd)
            }
          }
        }
      }
    }
    waitingForVisit.push(stage.rdd)
    while (waitingForVisit.nonEmpty) {
      visit(waitingForVisit.pop())
    }
    missing.toList
  }
~~~

###submitMissingTasks函数     
可见无论是哪种stage，都是对于每个stage中的每个partitions创建task，并最终封装成TaskSet，将该stage提交给taskscheduler。
~~~ scala
  /** Called when stage's parents are available and we can now do its task. */
  private def submitMissingTasks(stage: Stage, jobId: Int) {
    logDebug("submitMissingTasks(" + stage + ")")
    // Get our pending tasks and remember them in our pendingTasks entry
    stage.pendingTasks.clear()

    // First figure out the indexes of partition ids to compute.
    val (allPartitions: Seq[Int], partitionsToCompute: Seq[Int]) = {
      stage match {
        case stage: ShuffleMapStage =>
          val allPartitions = 0 until stage.numPartitions
          val filteredPartitions = allPartitions.filter { id => stage.outputLocs(id).isEmpty }
          (allPartitions, filteredPartitions)
        case stage: ResultStage =>
          val job = stage.resultOfJob.get
          val allPartitions = 0 until job.numPartitions
          val filteredPartitions = allPartitions.filter { id => !job.finished(id) }
          (allPartitions, filteredPartitions)
      }
    }

    // Create internal accumulators if the stage has no accumulators initialized.
    // Reset internal accumulators only if this stage is not partially submitted
    // Otherwise, we may override existing accumulator values from some tasks
    if (stage.internalAccumulators.isEmpty || allPartitions == partitionsToCompute) {
      stage.resetInternalAccumulators()
    }

    val properties = jobIdToActiveJob.get(stage.firstJobId).map(_.properties).orNull

    runningStages += stage
    // SparkListenerStageSubmitted should be posted before testing whether tasks are
    // serializable. If tasks are not serializable, a SparkListenerStageCompleted event
    // will be posted, which should always come after a corresponding SparkListenerStageSubmitted
    // event.
    outputCommitCoordinator.stageStart(stage.id)
    val taskIdToLocations = try {
      stage match {
        case s: ShuffleMapStage =>
          partitionsToCompute.map { id => (id, getPreferredLocs(stage.rdd, id))}.toMap
        case s: ResultStage =>
          val job = s.resultOfJob.get
          partitionsToCompute.map { id =>
            val p = job.partitions(id)
            (id, getPreferredLocs(stage.rdd, p))
          }.toMap
      }
    } catch {
      case NonFatal(e) =>
        stage.makeNewStageAttempt(partitionsToCompute.size)
        listenerBus.post(SparkListenerStageSubmitted(stage.latestInfo, properties))
        abortStage(stage, s"Task creation failed: $e\n${e.getStackTraceString}", Some(e))
        runningStages -= stage
        return
    }

    stage.makeNewStageAttempt(partitionsToCompute.size, taskIdToLocations.values.toSeq)
    listenerBus.post(SparkListenerStageSubmitted(stage.latestInfo, properties))

    // TODO: Maybe we can keep the taskBinary in Stage to avoid serializing it multiple times.
    // Broadcasted binary for the task, used to dispatch tasks to executors. Note that we broadcast
    // the serialized copy of the RDD and for each task we will deserialize it, which means each
    // task gets a different copy of the RDD. This provides stronger isolation between tasks that
    // might modify state of objects referenced in their closures. This is necessary in Hadoop
    // where the JobConf/Configuration object is not thread-safe.
    var taskBinary: Broadcast[Array[Byte]] = null
    try {
      // For ShuffleMapTask, serialize and broadcast (rdd, shuffleDep).
      // For ResultTask, serialize and broadcast (rdd, func).
      val taskBinaryBytes: Array[Byte] = stage match {
        case stage: ShuffleMapStage =>
          closureSerializer.serialize((stage.rdd, stage.shuffleDep): AnyRef).array()
        case stage: ResultStage =>
          closureSerializer.serialize((stage.rdd, stage.resultOfJob.get.func): AnyRef).array()
      }

      taskBinary = sc.broadcast(taskBinaryBytes)
    } catch {
      // In the case of a failure during serialization, abort the stage.
      case e: NotSerializableException =>
        abortStage(stage, "Task not serializable: " + e.toString, Some(e))
        runningStages -= stage

        // Abort execution
        return
      case NonFatal(e) =>
        abortStage(stage, s"Task serialization failed: $e\n${e.getStackTraceString}", Some(e))
        runningStages -= stage
        return
    }

    val tasks: Seq[Task[_]] = try {
      stage match {
        case stage: ShuffleMapStage =>
          partitionsToCompute.map { id =>
            val locs = taskIdToLocations(id)
            val part = stage.rdd.partitions(id)
            new ShuffleMapTask(stage.id, stage.latestInfo.attemptId,
              taskBinary, part, locs, stage.internalAccumulators)
          }

        case stage: ResultStage =>
          val job = stage.resultOfJob.get
          partitionsToCompute.map { id =>
            val p: Int = job.partitions(id)
            val part = stage.rdd.partitions(p)
            val locs = taskIdToLocations(id)
            new ResultTask(stage.id, stage.latestInfo.attemptId,
              taskBinary, part, locs, id, stage.internalAccumulators)
          }
      }
    } catch {
      case NonFatal(e) =>
        abortStage(stage, s"Task creation failed: $e\n${e.getStackTraceString}", Some(e))
        runningStages -= stage
        return
    }

    if (tasks.size > 0) {
      logInfo("Submitting " + tasks.size + " missing tasks from " + stage + " (" + stage.rdd + ")")
      stage.pendingTasks ++= tasks
      logDebug("New pending tasks: " + stage.pendingTasks)
      taskScheduler.submitTasks(new TaskSet(
        tasks.toArray, stage.id, stage.latestInfo.attemptId, stage.firstJobId, properties))
      stage.latestInfo.submissionTime = Some(clock.getTimeMillis())
    } else {
      // Because we posted SparkListenerStageSubmitted earlier, we should mark
      // the stage as completed here in case there are no tasks to run
      markStageAsFinished(stage, None)

      val debugString = stage match {
        case stage: ShuffleMapStage =>
          s"Stage ${stage} is actually done; " +
            s"(available: ${stage.isAvailable}," +
            s"available outputs: ${stage.numAvailableOutputs}," +
            s"partitions: ${stage.numPartitions})"
        case stage : ResultStage =>
          s"Stage ${stage} is actually done; (partitions: ${stage.numPartitions})"
      }
      logDebug(debugString)
    }
  }
~~~
