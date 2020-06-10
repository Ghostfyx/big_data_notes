# SparkContext源码分析

## 1. SparkContext简介

Spark应用程序的第一步就是创建并初始化SparkContext，SparkContext的初始化过程包含了内部组件的创建和准备，主要涉及网络通信、分布式、消息、存储、计算、调度、缓存、度量、清理、文件服务和UI等方面。它是Spark主要功能的入口点，链接Spark集群，创建RDD、累加器和广播变量，一个线程只能运行一个SparkContext。SparkContext在应用程序中将外部数据转换成RDD，因此建立了第一个RDD，也就是说SparkContext建立了RDD血缘关系的根，是DAG的根源。

## 2. SparkContext初始化过程

### 2.1 activeContext&&contextBeingConstructed

在将SparkContext初始化过程之前，需要先了解SparkContext伴生对象中的两个变量，它们分别是activeContext: AtomicReference[SparkContext]和contextBeingConstructed: Option[SparkContext]。

activeContext: AtomicReference[SparkContext]记录了当前SparkContext是否处于活跃状态，当活跃的时候activeContext的value就是当前SparkContext，否则value就是null。

contextBeingConstructed: Option[SparkContext]则是SparkContext正在启动时的一个标识，SparkContext初始化时有很多组件需要进行初始化设置，需要消耗一些时间，同时又要保证一个线程中只运行一个SparkContext，通过设置SparkContext启动时的表示，来保证一个线程中只运行一个SparkContext，当SparkContext正在启动时，contextBeingConstructed=Some(sc)，否则contextBeingConstructed=None。

源码如下：

```scala
/**
   * The active, fully-constructed SparkContext.  If no SparkContext is active, then this is `null`.
   * 正在运行的，完全构建完成的SparkContext
   *
   * Access to this field is guarded by SPARK_CONTEXT_CONSTRUCTOR_LOCK.
   * SPARK_CONTEXT_CONSTRUCTOR_LOCK保护对该字段的访问。
   */
private val activeContext: AtomicReference[SparkContext] =
new AtomicReference[SparkContext](null)

/**
   * Points to a partially-constructed SparkContext if some thread is in the SparkContext
   * constructor, or `None` if no SparkContext is being constructed.
   * 如果某个线程在SparkContext构造函数中，则指向部分构造的SparkContext；如果没有正在构造的SparkContext，则指向“ None”。
   * Access to this field is guarded by SPARK_CONTEXT_CONSTRUCTOR_LOCK
   */
private var contextBeingConstructed: Option[SparkContext] = None
```

### 2.2  初始化第一步：markPartiallyConstructed方法

```scala
// In order to prevent multiple SparkContexts from being active at the same time, mark this
// context as having started construction.
// NOTE: this must be placed at the beginning of the SparkContext constructor.
// 为了防止同一时间内，在一个线程中同时创建多个SparkContexts，确保当前线程中没有SparkContext在运行，
// 因此会在SparkContexts创建一开始时调用
SparkContext.markPartiallyConstructed(this, allowMultipleContexts)

/**
   * SparkContext构造器初始化的第一步：确保当前线程中没有SparkContext在运行。如果当前线程有SparkContext
   *
   * Called at the beginning of the SparkContext constructor to ensure that no SparkContext is
   * running.  Throws an exception if a running context is detected and logs a warning if another
   * thread is constructing a SparkContext.  This warning is necessary because the current locking
   * scheme prevents us from reliably distinguishing between cases where another context is being
   * constructed and cases where another constructor threw an exception.
   */
private[spark] def markPartiallyConstructed(
    sc: SparkContext,
    allowMultipleContexts: Boolean): Unit = {
    SPARK_CONTEXT_CONSTRUCTOR_LOCK.synchronized {
        assertNoOtherContextIsRunning(sc, allowMultipleContexts)
        // 若assertNoOtherContextIsRunning没有抛出异常，
        // 设置contextBeingConstructed = Some(sc)，表示当前线程中正在建立SparkContext
        contextBeingConstructed = Some(sc)
    }
}

/**
   * Called to ensure that no other SparkContext is running in this JVM.
   * 判断当前JVM中没有其他SparkContext正在运行
   *
   * Throws an exception if a running context is detected and logs a warning if another thread is
   * constructing a SparkContext.  This warning is necessary because the current locking scheme
   * prevents us from reliably distinguishing between cases where another context is being
   * constructed and cases where another constructor threw an exception.
   *如果检测到正在运行的SparkContext则抛出异常；如果其他线程正在构建SparkContext则打印一个警告，为了区分不同上下文抛出的异常
   *
   */
private def assertNoOtherContextIsRunning(
    sc: SparkContext,
    allowMultipleContexts: Boolean): Unit = {
    SPARK_CONTEXT_CONSTRUCTOR_LOCK.synchronized {
        Option(activeContext.get()).filter(_ ne sc).foreach { ctx =>
            val errMsg = "Only one SparkContext may be running in this JVM (see SPARK-2243)." +
            " To ignore this error, set spark.driver.allowMultipleContexts = true. " +
            s"The currently running SparkContext was created at:\n${ctx.creationSite.longForm}"
            val exception = new SparkException(errMsg)
            if (allowMultipleContexts) {
                logWarning("Multiple running SparkContexts detected in the same JVM!", exception)
            } else {
                throw exception
            }
        }

        contextBeingConstructed.filter(_ ne sc).foreach { otherContext =>
            // Since otherContext might point to a partially-constructed context, guard against
            // its creationSite field being null:
            val otherContextCreationSite =
            Option(otherContext.creationSite).map(_.longForm).getOrElse("unknown location")
            val warnMsg = "Another SparkContext is being constructed (or threw an exception in its" +
            " constructor).  This may indicate an error, since only one SparkContext may be" +
            " running in this JVM (see SPARK-2243)." +
            s" The other SparkContext was created at:\n$otherContextCreationSite"
            logWarning(warnMsg)
        }
    }
}
```

### 2.3 初始化第二步：打印版本号

```scala
// log out Spark Version in Spark driver log
logInfo(s"Running Spark version $SPARK_VERSION")
```

### 2.4 初始化第三步：正式开始初始化

初始化过程分为十几个步骤，主要分为三个大的阶段：

- 校验conf中的配置信息(部署模式、appName、yarn模式校验等等)
- 处理或设置参数
- 初始化启动一些组件

### 2.4.1 校验conf配置信息

```scala
// 通过克隆的方式获取sparkconf
_conf = config.clone()
// 对conf中的配置信息进行校验（部署模式、appName、yarn模式校验等等）
_conf.validateSettings()

// 部署模式：spark.master必填
if (!_conf.contains("spark.master")) {
    throw new SparkException("A master URL must be set in your configuration")
}

// 应用程序名称必填：spark.app.name
if (!_conf.contains("spark.app.name")) {
    throw new SparkException("An application name must be set in your configuration")
}

// log out spark.app.name in the Spark driver logs
logInfo(s"Submitted application: $appName")

// System property spark.yarn.app.id must be set if user code ran by AM on a YARN cluster
// 如果使用Yarn Cluster方式，spark.yarn.app.id必填
if (master == "yarn" && deployMode == "cluster" && !_conf.contains("spark.yarn.app.id")) {
    throw new SparkException("Detected yarn cluster mode, but isn't running on a cluster. " +
                             "Deployment to YARN is not supported directly by SparkContext. Please use spark-submit.")
}

if (_conf.getBoolean("spark.logConf", false)) {
    logInfo("Spark configuration:\n" + _conf.toDebugString)
}
```

### 2.4.2 处理或设置参数

```scala
// Set Spark driver host and port system properties. This explicitly sets the configuration
// instead of relying on the default value of the config constant.
// 设置driver的IP、端口号
_conf.set(DRIVER_HOST_ADDRESS, _conf.get(DRIVER_HOST_ADDRESS))
_conf.setIfMissing("spark.driver.port", "0")

// 设置Driver ID
_conf.set("spark.executor.id", SparkContext.DRIVER_IDENTIFIER)

// 处理jar路径和文件路径
_jars = Utils.getUserJars(_conf)
_files = _conf.getOption("spark.files").map(_.split(",")).map(_.filter(_.nonEmpty))
.toSeq.flatten

// 设置事件日志路径、是否压缩事件
_eventLogDir =
if (isEventLogEnabled) {
    val unresolvedDir = conf.get("spark.eventLog.dir", EventLoggingListener.DEFAULT_LOG_DIR)
    .stripSuffix("/")
    Some(Utils.resolveURI(unresolvedDir))
} else {
    None
}

_eventLogCodec = {
    val compress = _conf.getBoolean("spark.eventLog.compress", false)
    if (compress && isEventLogEnabled) {
        Some(CompressionCodec.getCodecName(_conf)).map(CompressionCodec.getShortName)
    } else {
        None
    }
}
```

### 2.4.3 初始化一些组件(重点)

几个要的核心组件如下：

**LiveListenerBus**

```java
// 创建事件总线
 _listenerBus = new LiveListenerBus(_conf)
```

**AppStatusStore**

```java
// 创建任务进度监听器,并增加到事件总线中
_statusStore = AppStatusStore.createLiveStore(conf)
listenerBus.addToStatusQueue(_statusStore.listener.get)
```

**SparkEnv**

```java
// 创建Spark运行环境
_env = createSparkEnv(_conf, isLocal, listenerBus)
SparkEnv.set(_env)
   
// This function allows components created by SparkEnv to be mocked in unit tests:
private[spark] def createSparkEnv(
  conf: SparkConf,
  isLocal: Boolean,
  listenerBus: LiveListenerBus): SparkEnv = {
  // 创建Spark Driver运行环境
  SparkEnv.createDriverEnv(conf, isLocal, listenerBus,    SparkContext.numDriverCores(master, conf))
}    
```

**SparkStatusTracker**

```scala
// 创建状态跟踪器
_statusTracker = new SparkStatusTracker(this, _statusStore)
```

**progressBar**

```scala
// 创建进度条，在控制台显示进度条
_progressBar =
if (_conf.get(UI_SHOW_CONSOLE_PROGRESS) && !log.isInfoEnabled) {
    Some(new ConsoleProgressBar(this))
} else {
    None
}
```

**SparkUI**

```scala
 _ui =
      // 默认情况下启动应用程序的UI监控，在监控过程中加入把作业处理监听器JobProgressListener
      // 注入到消息总线ListenerBus中，用于监控作业处理状态
      if (conf.get(UI_ENABLED)) {
        Some(SparkUI.create(Some(this), _statusStore, _conf, _env.securityManager, appName, "",
          startTime))
      } else {
        // For tests, do not enable the UI
        None
      }
    // Bind the UI before starting the task scheduler to communicate
    // the bound port to the cluster manager properly
    // 如果端口被占用就会逐步递增，默认端口是4040
    _ui.foreach(_.bind())
```

**HadoopConfiguration**

创建hadoop的配置信息（SPARK_YARN_MODE=true时，采用yarn配置信息）

```scala
_hadoopConfiguration = SparkHadoopUtil.get.newConfiguration(_conf)
```

**Load Jar && Files**

```scala
// Add each JAR given through the constructor
if (jars != null) {
    jars.foreach(addJar)
}

if (files != null) {
    files.foreach(addFile)
}
```

**Executor运行环境**

创建心跳接收器，在创建taskScheduler之前创建，因为Executor需要再构造函数中检索heartbeatReceiver

```scala
_heartbeatReceiver = env.rpcEnv.setupEndpoint(
    HeartbeatReceiver.ENDPOINT_NAME, new HeartbeatReceiver(this))
```

**HeartbeatReceiver**

```scala
// We need to register "HeartbeatReceiver" before "createTaskScheduler" because Executor will
// retrieve "HeartbeatReceiver" in the constructor. (SPARK-6640)
// 在创建TaskScheduler之前注册HeartbeatReceiver，因为Executor需要再构造函数中检索heartbeatReceiver
_heartbeatReceiver = env.rpcEnv.setupEndpoint(
    HeartbeatReceiver.ENDPOINT_NAME, new HeartbeatReceiver(this))
```

**schedulerBackend && taskScheduler**

```scala
// Create and start the scheduler
val (sched, ts) = SparkContext.createTaskScheduler(this, master, deployMode)
_schedulerBackend = sched
_taskScheduler = ts
```

**DAGScheduler**

向dagScheduler引入了taskScheduler

```scala
_dagScheduler = new DAGScheduler(this)
_heartbeatReceiver.ask[Boolean](TaskSchedulerIsSet)
```

**启动TaskScheduler**

启动taskScheduler，并根据taskScheduler生成的_applicationId启动度量系统，并且将监控信息发送给SparkUI进行展示

```scala
// start TaskScheduler after taskScheduler sets DAGScheduler reference in DAGScheduler's
// constructor
// 启动TaskScheduler
_taskScheduler.start()

// 当前应用程序ID
_applicationId = _taskScheduler.applicationId()
// 当前应用程序尝试ID，第一次尝试执行任务_applicationAttemptId=1，第二次尝试执行任务_applicationAttemptId=2
_applicationAttemptId = taskScheduler.applicationAttemptId()
_conf.set("spark.app.id", _applicationId)
if (_conf.getBoolean("spark.ui.reverseProxy", false)) {
    System.setProperty("spark.ui.proxyBase", "/proxy/" + _applicationId)
}
// Spark UI 中根据应用程序ID展示应用程序信息
_ui.foreach(_.setAppId(_applicationId))
// 块管理器根据应用程序做初始化工作
_env.blockManager.initialize(_applicationId)

// The metrics system for Driver need to be set spark.app.id to app ID.
// So it should start after we get app ID from the task scheduler and set spark.app.id.
// 根据app ID，启动度量系统并且将运行情况发送给Spark UI
_env.metricsSystem.start()
// Attach the driver metrics servlet handler to the web ui after the metrics system is started.
_env.metricsSystem.getServletHandlers.foreach(handler => ui.foreach(_.attachHandler(handler)))
```

**EventLoggingListener**

```scala
// 建立事件日志的监听器，并添加到事件总线中
_eventLogger =
if (isEventLogEnabled) {
    val logger =
    new EventLoggingListener(_applicationId, _applicationAttemptId, _eventLogDir.get,
                             _conf, _hadoopConfiguration)
    logger.start()
    listenerBus.addToEventLogQueue(logger)
    Some(logger)
} else {
    None
}
```

**ExecutorAllocationManager**

创建并启动Executor动态分配管理器

```scala
// Optionally scale number of executors dynamically based on workload. Exposed for testing.
val dynamicAllocationEnabled = Utils.isDynamicAllocationEnabled(_conf)
_executorAllocationManager =
if (dynamicAllocationEnabled) {
    schedulerBackend match {
        case b: ExecutorAllocationClient =>
        Some(new ExecutorAllocationManager(
            schedulerBackend.asInstanceOf[ExecutorAllocationClient], listenerBus, _conf,
            _env.blockManager.master))
        case _ =>
        None
    }
} else {
    None
}
_executorAllocationManager.foreach(_.start())
```

**ContextCleaner**

创建并启动上下文清理器

```scala
_cleaner =
if (_conf.getBoolean("spark.cleaner.referenceTracking", true)) {
    Some(new ContextCleaner(this))
} else {
    None
}
_cleaner.foreach(_.start())
```

**setupAndStartListenerBus**

设置并启动时间总线

```scala
setupAndStartListenerBus()
```

**postEnvironmentUpdate**

发布环境更新事件

```scala
postEnvironmentUpdate()
```

**postApplicationStart**

发布应用程序启动事件

```scala
postApplicationStart()
```

**metricsSystem.registerSource**

将dagScheduler、BlockManagerSource和ExecutorAllocationManager注册到度量系统中

```scala
_env.metricsSystem.registerSource(_dagScheduler.metricsSource)
_env.metricsSystem.registerSource(new BlockManagerSource(_env.blockManager))
_executorAllocationManager.foreach { e =>
    _env.metricsSystem.registerSource(e.executorAllocationManagerSource)
}
```

### 2.5 初始化四：确认启动成功

```scala
// In order to prevent multiple SparkContexts from being active at the same time, mark this
// context as having finished construction.
// NOTE: this must be placed at the end of the SparkContext constructor.
// 为了防止同时激活多个SparkContext，请将此上下文标记为已完成构造。
// 注意：这必须在SparkContext构造器结尾
SparkContext.setActiveContext(this, allowMultipleContexts)
```

## 3. SparkContext 提交运行Job

### 3.1 runJob方法

SparkContext的runJob方法，其中调用了多层runJob方法。

```scala
/**
   * Run a job on all partitions in an RDD and return the results in an array.
   *
   * 在RDD中的所有分区上运行作业，然后将结果返回到数组中。
   *
   * @param rdd target RDD to run tasks on
   * @param func a function to run on each partition of the RDD
   * @return in-memory collection with a result of the job (each collection element will contain
   * a result from one partition)
   */
def runJob[T, U: ClassTag](rdd: RDD[T], func: Iterator[T] => U): Array[U] = {
    runJob(rdd, func, 0 until rdd.partitions.length)
}
```

第一层runJob

```scala
/**
   * Run a function on a given set of partitions in an RDD and return the results as an array.
   *
   * @param rdd target RDD to run tasks on
   * @param func a function to run on each partition of the RDD
   * @param partitions set of partitions to run on; some jobs may not want to compute on all
   * partitions of the target RDD, e.g. for operations like `first()`
   * @return in-memory collection with a result of the job (each collection element will contain
   * a result from one partition)
   */
def runJob[T, U: ClassTag](
    rdd: RDD[T],
    func: Iterator[T] => U,
    partitions: Seq[Int]): Array[U] = {
    val cleanedFunc = clean(func)
    runJob(rdd, (ctx: TaskContext, it: Iterator[T]) => cleanedFunc(it), partitions)
}
```

第二层runJob

```scala
/**
   * Run a function on a given set of partitions in an RDD and return the results as an array.
   * The function that is run against each partition additionally takes `TaskContext` argument.
   *
   * 针对每个分区运行的函数还带有`TaskContext`参数。
   * @param rdd target RDD to run tasks on
   * @param func a function to run on each partition of the RDD
   * @param partitions set of partitions to run on; some jobs may not want to compute on all
   * partitions of the target RDD, e.g. for operations like `first()`
   * @return in-memory collection with a result of the job (each collection element will contain
   * a result from one partition)
   */
def runJob[T, U: ClassTag](
    rdd: RDD[T],
    func: (TaskContext, Iterator[T]) => U,
    partitions: Seq[Int]): Array[U] = {
    val results = new Array[U](partitions.size)
    // 针对每个分区运行的函数还带有`TaskContext`参数。
    runJob[T, U](rdd, func, partitions, (index, res) => results(index) = res)
    results
}
```

第三层runJob，底层调用DAGScheduler的runJob方法

```scala
/**
   * Run a function on a given set of partitions in an RDD and pass the results to the given
   * handler function. This is the main entry point for all actions in Spark.
   *
   * 在一个RDD的给定分区上运行function，将结果传递给handler function，这是Spark中所有动作的主要切入点
   *
   * @param rdd target RDD to run tasks on
   * @param func a function to run on each partition of the RDD
   * @param partitions set of partitions to run on; some jobs may not want to compute on all
   * partitions of the target RDD, e.g. for operations like `first()`
   * @param resultHandler callback to pass each result to
   */
def runJob[T, U: ClassTag](
    rdd: RDD[T],
    func: (TaskContext, Iterator[T]) => U,
    partitions: Seq[Int],
    resultHandler: (Int, U) => Unit): Unit = {
    if (stopped.get()) {
        throw new IllegalStateException("SparkContext has been shutdown")
    }
    val callSite = getCallSite
    // 清理SparkContext，使其可以序列化并发送给任务
    val cleanedFunc = clean(func)
    logInfo("Starting job: " + callSite.shortForm)
    if (conf.getBoolean("spark.logLineage", false)) {
        logInfo("RDD's recursive dependencies:\n" + rdd.toDebugString)
    }
    // 启动dagScheduler，在SparkContext初始化时就已经构建了DAGScheduler
    dagScheduler.runJob(rdd, cleanedFunc, partitions, callSite, resultHandler, localProperties.get)
    // 进度条
    progressBar.foreach(_.finishAll())
    rdd.doCheckpoint()
}
```

