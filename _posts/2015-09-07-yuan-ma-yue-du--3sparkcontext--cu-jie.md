---
layout: post
title: "源码阅读 （3）SparkContext 粗解"
description: ""
category: ""
tags: [spark]
---
{% include JB/setup %}

SparkContext是应用启动时创建的Spark上下文对象，是Spark上层应用与底层实现之间的中介(SparkContext负责给executors发送task)。
主要涉及：

> SparkEnv    
> DAGScheduler     
> TaskScheduler     
> SchedulerBackend     
> SparkUI    

SparkContext的构造函数中最重要的参数是SparkConf。
SparkConf是通过一个HashMap来管理<key, value>类型的属性。

~~~ scala
private val settings = new ConcurrentHashMap[String, String]()
~~~

###创建LiveListenerBus监听器

~~~ scala
private[spark] val listenerBus = new LiveListenerBus
~~~

它是典型的观察者模式。

####创建SparkEnv运行环境


~~~ scala
private[spark] def createSparkEnv(

    conf: SparkConf,
    isLocal: Boolean,
    listenerBus: LiveListenerBus): SparkEnv = {
  SparkEnv.createDriverEnv(conf, isLocal, listenerBus)
}
~~~


在createDriverEnv中创建了MapOutputTracker,BlockManager,CacheManager,HttpFilerServer等一系列对象。


~~~ scala
val envInstance = new SparkEnv(

  executorId,
  rpcEnv,
  serializer,
  closureSerializer,
  cacheManager,
  mapOutputTracker,
  shuffleManager,
  broadcastManager,
  blockTransferService,
  blockManager,
  securityManager,
  httpFileServer,
  sparkFilesDir,
  metricsSystem,
  shuffleMemoryManager,
  executorMemoryManager,
  outputCommitCoordinator,
  conf)

~~~

其中：

> cacheManager:用于存储中间结果     
> mapOutputTracker:用来缓存MapStatus信息，并提供MapOutputMaster获取的功能    
> shuffleManager:路由维护表    
> broadcastManager:广播    
> blockManager:块管理    
> securityManager:安全管理    
> httpFileServer:文件存储服务器    
>   > sparkFilerDir:文件存储目录     
> metricsSystem:测量    
> conf:配置文件    

###创建SparkUI


~~~ scala
ui =

  if (conf.getBoolean("spark.ui.enabled", true)) {
    Some(SparkUI.createLiveUI(this, _conf, listenerBus, _jobProgressListener,
      _env.securityManager, appName, startTime = startTime))
  } else {
    // For tests, do not enable the UI
    None
  }
~~~


创建SparkUI时会向listenerBus注册StorageStatusListener，它负责监听Storage的变化及时的展示到Spark Web上。


~~~ scala
def initialize() {

  attachTab(new JobsTab(this))
  attachTab(stagesTab)
  attachTab(new StorageTab(this))
  attachTab(new EnvironmentTab(this))
  attachTab(new ExecutorsTab(this))
  attachHandler(createStaticHandler(SparkUI.STATIC_RESOURCE_DIR, "/static"))
  attachHandler(createRedirectHandler("/", "/jobs/", basePath = basePath))
  attachHandler(ApiRootResource.getServletHandler(this))
  // This should be POST only, but, the YARN AM proxy won't proxy POSTs
  attachHandler(createRedirectHandler(
    "/stages/stage/kill", "/stages/", stagesTab.handleKillRequest,
    httpMethods = Set("GET", "POST")))
}
~~~


attachTab方法中添加对象是我们在Spark Web页面中看到的那个标签。

###创建TaskScheduler


~~~ scala
val (sched, ts) = SparkContext.createTaskScheduler(this, master)

_schedulerBackend = sched
_taskScheduler = ts
~~~


createTaskScheduler会根据不同的master url创建不同的schedulerBackend和taskScheduler。他们在后续的task分发过程中扮演重要角色。

###创建DAGScheduler

~~~ scala
_dagScheduler = new DAGScheduler(this)
~~~


DAGScheduler构造函数需要上面创建的taskScheduler。
之后就会使用

~~~ scala
_taskScheduler.start()
~~~

启动相应的SchedulerBackend,并启动定时器进行检测：

~~~ scala
override def start() {

  backend.start()

  if (!isLocal && conf.getBoolean("spark.speculation", false)) {
    logInfo("Starting speculative execution thread")
    speculationScheduler.scheduleAtFixedRate(new Runnable {
      override def run(): Unit = Utils.tryOrStopSparkContext(sc) {
        checkSpeculatableTasks()
      }
    }, SPECULATION_INTERVAL_MS, SPECULATION_INTERVAL_MS, TimeUnit.MILLISECONDS)
  }
}
~~~


启动listenerBus并发布 SparkListenerEnvironmentUpdate 和 SparkListenerApplicationStart 这两个事件。对这两个事件，监听器会调用onEnvironmentUpdate、onApplicationStart方法进行处理。

###RunJob
主要调用

~~~ scala
dagScheduler.runJob(rdd, cleanedFunc, partitions, callSite, resultHandler, localProperties.get)
~~~

来执行工作。
