---
layout: post
title: "源码阅读（6）任务执行"
description: ""
category: ""
tags: []
---
{% include JB/setup %}

### Task ### 

Spark有两种Task:    
 1. ShuffleMapTask    
 2. ResultTask

Task是Spark的Executor执行单元，介于DAGScheduler和TaskScheduler中间的接口。在DAGScheduler中，需要把DAG中的每个stage的每个partitions封装成task，再把这些taskset提交给TaskScheduler。    

~~~ scala
private[spark] abstract class Task[T](
    val stageId: Int,  //这个task属于那个stage
    val stageAttemptId: Int,
    val partitionId: Int,  //RDD中的序号
    internalAccumulators: Seq[Accumulator[Long]]) extends Serializable
~~~

由于Task需要保证工作节点具备本次Task需要的其他依赖注册到SparkContext下，所以Task的伴生对象提供了序列化和反序列化应用依赖的jar包的方法，来写入写出依赖流。

~~~ scala
private[spark] object Task {
  /**
   * Serialize a task and the current app dependencies (files and JARs added to the SparkContext)
   */
  def serializeWithDependencies(
  task: Task[_],
      currentFiles: HashMap[String, Long],
      currentJars: HashMap[String, Long],
      serializer: SerializerInstance)
    : ByteBuffer = {


  def deserializeWithDependencies(serializedTask: ByteBuffer)

~~~

#### ShuffleMapTask ####   

对应于ShuffleMap Stage, 产生的结果作为其他stage的输入。

~~~ scala
/**
* A ShuffleMapTask divides the elements of an RDD into multiple buckets (based on a partitioner
* specified in the ShuffleDependency).
*
* See [[org.apache.spark.scheduler.Task]] for more information.
*
 * @param stageId id of the stage this task belongs to
 * @param taskBinary broadcast version of the RDD and the ShuffleDependency. Once deserialized,
 *                   the type should be (RDD[_], ShuffleDependency[_, _, _]).
 * @param partition partition of the RDD this task is associated with
 * @param locs preferred task execution locations for locality scheduling
 */
private[spark] class ShuffleMapTask(
    stageId: Int,
    stageAttemptId: Int,
    taskBinary: Broadcast[Array[Byte]],
    partition: Partition,
    @transient private var locs: Seq[TaskLocation],
    internalAccumulators: Seq[Accumulator[Long]])
  extends Task[MapStatus](stageId, stageAttemptId, partition.index, internalAccumulators)
  with Logging {
~~~

#### ResultTask ####   

对Result Stage直接产生结果    


~~~ scala
/**
 * A task that sends back the output to the driver application.
 *
 * See [[Task]] for more information.
 *
 * @param stageId id of the stage this task belongs to
 * @param taskBinary broadcasted version of the serialized RDD and the function to apply on each
 *                   partition of the given RDD. Once deserialized, the type should be
 *                   (RDD[T], (TaskContext, Iterator[T]) => U).
 * @param partition partition of the RDD this task is associated with
 * @param locs preferred task execution locations for locality scheduling
 * @param outputId index of the task in this job (a job can launch tasks on only a subset of the
 *                 input RDD's partitions).
 */
private[spark] class ResultTask[T, U](
    stageId: Int,
    stageAttemptId: Int,
    taskBinary: Broadcast[Array[Byte]],
    partition: Partition,
    @transient locs: Seq[TaskLocation],
    val outputId: Int,
    internalAccumulators: Seq[Accumulator[Long]])
  extends Task[U](stageId, stageAttemptId, partition.index, internalAccumulators)
  with Serializable {
~~~

#### TaskSet ####   
TaskSet是用于封装一个stage的所有的task的数据结构，用来方便的提交给TaskScheduler。    

~~~ scala
/**
 * A set of tasks submitted together to the low-level TaskScheduler, usually representing
 * missing partitions of a particular stage.
 */
private[spark] class TaskSet(
    val tasks: Array[Task[_]],
    val stageId: Int,
    val stageAttemptId: Int,
    val priority: Int,
    val properties: Properties) {
    val id: String = stageId + "." + stageAttemptId

  override def toString: String = "TaskSet " + id
}
~~~


### Executor注册到Driver ###   

Driver发送LaunchTask消息被Executor接收，Executor会使用launchTask对消息进行处理。
不过在这个过程之前，如果Executor没有注册到Driver，即便接收到LaunchTask指令，也不会做任务处理。所以我们要先搞清楚，Executor是如何在Driver侧进行注册的。

#### Application注册 ####   
Executor的注册是发生在Application的注册过程中的，我们以Standalone模式为例：    
1. SparkContext创建schedulerBackend和taskScheduler，schedulerBackend作为TaskScheduler对象的一个成员存在     
2. 在TaskScheduler对象调用start函数时，其实调用了backend.start()函数    
3. backend.start()函数中启动了AppClient，AppClient的其中一个参数ApplicationDescription就是封装的运行CoarseGrainedExecutorBackend的命令
4. AppClient内部启动了一个ClientActor，这个ClientActor启动之后，会尝试向Master发送一个指令actor ! RegisterApplication(appDescription) 注册一个Application

##### AppClient向Master提交Application #####   

