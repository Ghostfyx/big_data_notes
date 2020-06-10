# DAGScheduler源码分析

## 1. DAGScheduler简介

### 1.1 源码类注释

实现面向阶段调度的高级调度层。它为每个作业计算阶段的DAG(有向无环图)，跟踪实现了哪些RDD和阶段输出，并找到运行该作业的最小计划。 然后，它将阶段作为TaskSet提交给在群集上运行它们的基础TaskScheduler实现。 TaskSet包含完全独立的任务，这些任务可以基于集群中已经存在的数据(例如：先前阶段的map输出文件)。

Spark通过在RDD的Shuffle边界划分阶段，具有窄依赖关系的的RDD操作(例如map()和filter() )在每个阶段都通过管道传递到一组任务中。但是具有shuffle依赖的操作需要多个阶段(其一将前一个阶段的输出结果写入到map输出文件中，其二在阶段屏障后读取之前的输出文件)。最后，所有阶段之间只有shuffle依赖，而且Stage内可以计算多个算子，这些操作的实际流水线发生在各种RDD的RDD.compute()函数中。

除了提出阶段的DAG之外，DAGScheduler还根据当前缓存状态确定运行每个任务的首选位置，并将这些位置传递给低级TaskScheduler。 此外，它可以处理由于混洗输出文件丢失而导致的故障，在这种情况下，可能需要重新提交旧阶段。 一个阶段中不是由随机文件丢失引起的故障由TaskScheduler处理，该任务将在取消整个阶段之前重试每个任务几次。

在阅读源码的时候，有以下几个关键组件：

- **Jobs(由ActiveJob表示)**：是提交给scheduler的顶级工作项。例如：当用户调用action算子，像count()，将会通过submitJob提交作业。每个作业可能需要执行多个Stage才能构建中间数据。
- **Stage：**是计算作业中中间结果的任务集，其中每个任务在相同RDD的分区上计算相同功能。阶段在Shuffle边界划分，这样就会引入barrier，我们必须等待上一阶段完成。有两种类型的**Stage：ResultStage**——在执行动作的最后阶段；**ShuffleMapStage**——在shuffle阶段写入map输出文件。如果这些作业重复使用相同的RDD，则阶段通常在多个作业之间共享。
- **Task：**各个工作单元，每个Task发送到一台机器
- **Cache tracking**：DAGScheduler会计算出缓存了哪些RDD，以避免重新计算它们，并且同样记住哪些shuffle map阶段已经生成了输出文件，避免shuffle阶段重新执行map输出
- **Preferred locations：**DAGScheduler还会根据其基础RDD的首选位置、缓存或shuffle数据的位置，计算
- **Cleanup：**所有数据结构都会被清除，在依赖于它们的所有正在运行的作业完成后

要从故障中恢复，同一阶段可能需要运行多次，这称为**attempts**。如果TaskScheduler报告任务失败，原因是前一阶段的map输出文件丢失。DAGScheduler重新提交数据丢失的阶段，这是通过带有FetchFailed的CompletionEvent或ExecutorLost事件检测到的。DAGScheduler将等待一小段时间，以查看其他节点或任务是否失败，然后针对计算丢失任务的任何丢失阶段重新提交TaskSet。作为此过程的一部分，我们可能还必须为旧的(完成的)阶段创建Stage对象，在此之前我们曾清理过Stage对象。由于来自阶段的旧尝试的任务仍然可以运行，因此必须注意在正确的Stage对象中接收到的所有事件。

以下是在对此类进行更改或检查更改时要使用的清单：

- 当涉及它们的作业结束时，应清除所有数据结构，以避免在长时间运行的程序中无限期累积状态
- 添加新数据结构时，请更新DAGSchedulerSuite.assertDataStructuresEmpty从而包括新结构。 这将有助于捕获内存泄漏。

## 1.2 总结

Spark的调度阶段(Stage)是由DAGScheduler实现的，DAGScheduler会从最后一个RDD出发，使用广度优先遍历整个依赖树，从而划分调度阶段，调度阶段划分依据是以操作是否为宽依赖(ShuffleDependency)进行的，即当某个RDD的操作是Shuffle时，以该Shuffle操作为界限划分为前后两个调度阶段。

## 2. 源码详解

### 2.1 构建DAGScheduler

Spark在构造SparkContext时就会生成DAGScheduler的实例。

```scala
_dagScheduler = new DAGScheduler(this)//生成dagScheduler，传入当前sparkContext对象。
```

　在生成dagScheduler之前，已经生成了schedulerBackend和taskScheduler对象。之所以taskScheduler对象在dagScheduler对象构造之前先生成，是由于在生成DAGScheduler的构造方法中会从传入的SparkContext中获取到taskScheduler对象。

```scala
// SparkContext初始化时，调用创建DAGScheduler对象
def this(sc: SparkContext) = this(sc, sc.taskScheduler)

def this(sc: SparkContext, taskScheduler: TaskScheduler) = {
    this(
        sc,
        taskScheduler,
        sc.listenerBus,
        sc.env.mapOutputTracker.asInstanceOf[MapOutputTrackerMaster],
        sc.env.blockManager.master,
        sc.env)
}

// 主要构造方法
private[spark] class DAGScheduler(
    private[scheduler] val sc: SparkContext, // 获得当前SparkContext对象
    private[scheduler] val taskScheduler: TaskScheduler,  // 获得当前saprkContext内置的taskScheduler
    listenerBus: LiveListenerBus, // 异步处理事件的对象，从sc中获取
    mapOutputTracker: MapOutputTrackerMaster,//运行在Driver端管理shuffle map task的输出，从sc中获取
    blockManagerMaster: BlockManagerMaster, //运行在driver端，管理整个Job的Block信息，从sc中获取
    env: SparkEnv, // 从sc中获取
    clock: Clock = new SystemClock())
```

### 2.2 DAGScheduler数据结构

```scala
private[spark] val metricsSource: DAGSchedulerSource = new DAGSchedulerSource(this)

// 生成JobId
private[scheduler] val nextJobId = new AtomicInteger(0)
// 总的Job数
private[scheduler] def numTotalJobs: Int = nextJobId.get()
// 下一个StageId
private val nextStageId = new AtomicInteger(0)

// 记录某个job对应的包含的所有stage
private[scheduler] val jobIdToStageIds = new HashMap[Int, HashSet[Int]]
// 记录StageId对应的Stage
private[scheduler] val stageIdToStage = new HashMap[Int, Stage]
/**
   * Mapping from shuffle dependency ID to the ShuffleMapStage that will generate the data for
   * that dependency. Only includes stages that are part of currently running job (when the job(s)
   * that require the shuffle stage complete, the mapping will be removed, and the only record of
   * the shuffle data will be in the MapOutputTracker).
   *
   * 从shuffle依赖项ID映射到ShuffleMapStage，
    * 它将为该依赖项生成数据。
    * 仅包括属于当前正在运行的作业的阶段（当要求完成混洗阶段的作业完成时，将删除映射，并且混洗数据的唯一记录将在MapOutputTracker中）。
   */
// 记录每一个shuffle对应的ShuffleMapStage，key为shuffleId
private[scheduler] val shuffleIdToMapStage = new HashMap[Int, ShuffleMapStage]
// 记录处于Active状态的job，key为jobId, value为ActiveJob类型对象
private[scheduler] val jobIdToActiveJob = new HashMap[Int, ActiveJob]

// Stages we need to run whose parents aren't done
// 等待运行的Stage，一般这些是在等待Parent Stage运行完成才能开始
private[scheduler] val waitingStages = new HashSet[Stage]

// Stages we are running right now
// 处于Running状态的Stage
private[scheduler] val runningStages = new HashSet[Stage]

// Stages that must be resubmitted due to fetch failures
// 失败原因为fetch failures的Stage，并等待重新提交
private[scheduler] val failedStages = new HashSet[Stage]

// active状态的Job列表
private[scheduler] val activeJobs = new HashSet[ActiveJob]

/**
   * Contains the locations that each RDD's partitions are cached on.  This map's keys are RDD ids
   * and its values are arrays indexed by partition numbers. Each array value is the set of
   * locations where that RDD partition is cached.
   *
   * All accesses to this map should be guarded by synchronizing on it (see SPARK-4454).
   */
private val cacheLocs = new HashMap[Int, IndexedSeq[Seq[TaskLocation]]]

// For tracking failed nodes, we use the MapOutputTracker's epoch number, which is sent with
// every task. When we detect a node failing, we note the current epoch number and failed
// executor, increment it for new tasks, and use this to ignore stray ShuffleMapTask results.
//
// TODO: Garbage collect information about failure epochs when we know there are no more
//       stray messages to detect.
private val failedEpoch = new HashMap[String, Long]

private [scheduler] val outputCommitCoordinator = env.outputCommitCoordinator

// A closure serializer that we reuse.
// This is only safe because DAGScheduler runs in a single thread.
private val closureSerializer = SparkEnv.get.closureSerializer.newInstance()

/** If enabled, FetchFailed will not cause stage retry, in order to surface the problem. */
private val disallowStageRetryForTest = sc.getConf.getBoolean("spark.test.noStageRetry", false)

/**
   * Whether to unregister all the outputs on the host in condition that we receive a FetchFailure,
   * this is set default to false, which means, we only unregister the outputs related to the exact
   * executor(instead of the host) on a FetchFailure.
   */
private[scheduler] val unRegisterOutputOnHostOnFetchFailure =
sc.getConf.get(config.UNREGISTER_OUTPUT_ON_HOST_ON_FETCH_FAILURE)

/**
   * Number of consecutive stage attempts allowed before a stage is aborted.
   */
private[scheduler] val maxConsecutiveStageAttempts =
sc.getConf.getInt("spark.stage.maxConsecutiveAttempts",
                  DAGScheduler.DEFAULT_MAX_CONSECUTIVE_STAGE_ATTEMPTS)

/**
   * Number of max concurrent tasks check failures for each barrier job.
   */
private[scheduler] val barrierJobIdToNumTasksCheckFailures = new ConcurrentHashMap[Int, Int]

/**
   * Time in seconds to wait between a max concurrent tasks check failure and the next check.
   */
private val timeIntervalNumTasksCheck = sc.getConf
.get(config.BARRIER_MAX_CONCURRENT_TASKS_CHECK_INTERVAL)

/**
   * Max number of max concurrent tasks check failures allowed for a job before fail the job
   * submission.
   */
private val maxFailureNumTasksCheck = sc.getConf
.get(config.BARRIER_MAX_CONCURRENT_TASKS_CHECK_MAX_FAILURES)

private val messageScheduler =
ThreadUtils.newDaemonSingleThreadScheduledExecutor("dag-scheduler-message")

// 处理Scheduler事件的对象
private[spark] val eventProcessLoop = new DAGSchedulerEventProcessLoop(this)
taskScheduler.setDAGScheduler(this)
```

DAGScheduler构造完成，并初始化一个eventProcessLoop实例后，会调用其eventProcessLoop.start()方法，启动一个多线程，然后把各种event都提交到eventProcessLoop中。这个**eventProcessLoop比较重要**。

### 2.3 作业提交

### 2.3.1 runJob方法

　一个Job实际上是从RDD调用一个Action操作开始的，该Action操作最终会进入到`org.apache.spark.SparkContext.runJob()`方法中，在SparkContext中有多个重载的runJob方法，最终入口是下面这个：

```scala
/**
   * Run an action job on the given RDD and pass all the results to the resultHandler function as
   * they arrive.
   *
   * 在给定的RDD上运行操作作业，并将所有结果在到达时传递给resultHandler函数。
   *
   * @param rdd target RDD to run tasks on
   * @param func a function to run on each partition of the RDD
   * @param partitions set of partitions to run on; some jobs may not want to compute on all
   *   partitions of the target RDD, e.g. for operations like first()
   * @param callSite where in the user program this job was called
   * @param resultHandler callback to pass each result to
   * @param properties scheduler properties to attach to this job, e.g. fair scheduler pool name
   *
   * @note Throws `Exception` when the job fails
   */
def runJob[T, U](
    rdd: RDD[T],
    func: (TaskContext, Iterator[T]) => U,
    partitions: Seq[Int],
    callSite: CallSite,
    resultHandler: (Int, U) => Unit,
    properties: Properties): Unit = {
    val start = System.nanoTime
    // 得到一个JobWaiter实例来监听Job的执行情况。
    val waiter = submitJob(rdd, func, partitions, callSite, resultHandler, properties)
    ThreadUtils.awaitReady(waiter.completionFuture, Duration.Inf)
    // 根据Job的状态不同的处理方式
    waiter.completionFuture.value.get match {
        case scala.util.Success(_) =>
        logInfo("Job %d finished: %s, took %f s".format
                (waiter.jobId, callSite.shortForm, (System.nanoTime - start) / 1e9))
        case scala.util.Failure(exception) =>
        logInfo("Job %d failed: %s, took %f s".format
                (waiter.jobId, callSite.shortForm, (System.nanoTime - start) / 1e9))
        // SPARK-8644: Include user stack trace in exceptions coming from DAGScheduler.
        val callerStackTrace = Thread.currentThread().getStackTrace.tail
        exception.setStackTrace(exception.getStackTrace ++ callerStackTrace)
        throw exception
    }
}
```

### 2.3.2 submitJob方法

进入submitJob方法，首先会去检查rdd的分区信息，在确保rdd分区信息正确的情况下，给当前job生成一个jobId，nexJobId在刚构造出来时是从0开始编号的，在同一个SparkContext中，jobId会逐渐顺延。然后构造出一个JobWaiter对象返回给上一级调用函数。通过eventProcessLoop提交该任务。

```java
/**
   * Submit an action job to the scheduler.
   *
   * @param rdd target RDD to run tasks on
   * @param func a function to run on each partition of the RDD
   * @param partitions set of partitions to run on; some jobs may not want to compute on all
   *   partitions of the target RDD, e.g. for operations like first()
   * @param callSite where in the user program this job was called
   * @param resultHandler callback to pass each result to
   * @param properties scheduler properties to attach to this job, e.g. fair scheduler pool name
   *
   * @return a JobWaiter object that can be used to block until the job finishes executing
   *         or can be used to cancel the job.
   *
   * @throws IllegalArgumentException when partitions ids are illegal
   */
def submitJob[T, U](
    rdd: RDD[T],
    func: (TaskContext, Iterator[T]) => U,
    partitions: Seq[Int],
    callSite: CallSite,
    resultHandler: (Int, U) => Unit,
    properties: Properties): JobWaiter[U] = {
    // Check to make sure we are not launching a task on a partition that does not exist.
    val maxPartitions = rdd.partitions.length
        partitions.find(p => p >= maxPartitions || p < 0).foreach { p =>
        throw new IllegalArgumentException(
        "Attempting to access a non-existent partition: " + p + ". " +
        "Total number of partitions: " + maxPartitions)
                                                                  }

    // 供0开始递增获取Job ID，在同一个SparkContext中，jobId会逐渐顺延
    val jobId = nextJobId.getAndIncrement()
        if (partitions.size == 0) {
            // Return immediately if the job is running 0 tasks
            return new JobWaiter[U](this, jobId, 0, resultHandler)
        }

    assert(partitions.size > 0)
        val func2 = func.asInstanceOf[(TaskContext, Iterator[_]) => _]
        // 构造出一个JobWaiter对象返回给上一级调用函数
        val waiter = new JobWaiter(this, jobId, partitions.size, resultHandler)
        // 提交任务
        eventProcessLoop.post(JobSubmitted(
            jobId, rdd, func2, partitions.toArray, callSite, waiter,
            SerializationUtils.clone(properties)))
        waiter
}
```

### 2.3.3 DAGSchedulerEventProcessLoop post方法

```scala
private val eventQueue: BlockingQueue[E] = new LinkedBlockingDeque[E]() // 事件队列
def post(event: E): Unit = {
    eventQueue.put(event) // 将JobSubmitted，Job提交事件存入该队列中
}
```

### 2.3.4 EventLoop#run

```scala
override def run(): Unit = {
    try {
        while (!stopped.get) {
            // 事件出队
            val event = eventQueue.take()
            try {
                // 接收并处理事件
                onReceive(event)
            } catch {
                case NonFatal(e) =>
                try {
                    onError(e)
                } catch {
                    case NonFatal(e) => logError("Unexpected error in " + name, e)
                }
            }
        }
    } catch {
        case ie: InterruptedException => // exit even if eventQueue is not empty
        case NonFatal(e) => logError("Unexpected error in " + name, e)
    }
}
```

### 2.3.5 DAGSchedulerEventProcessLoop#onReceive

在onReceive方法中，进一步调用doOnReceive方法

```scala
**
* The main event loop of the DAG scheduler.
*/
override def onReceive(event: DAGSchedulerEvent): Unit = {
    val timerContext = timer.time()
    try {
        doOnReceive(event)
    } finally {
        timerContext.stop()
    }
}
```

### 2.3.6 DAGSchedulerEventProcessLoop#doOnReceive

　　在该方法中，根据事件类别分别匹配不同的方法进一步处理。本次传入的是JobSubmitted方法，那么进一步调用的方法是DAGScheduler.handleJobSubmitted。

```scala
private def doOnReceive(event: DAGSchedulerEvent): Unit = event match {
    // 处理Job提交事件
    case JobSubmitted(jobId, rdd, func, partitions, callSite, listener, properties) =>
    dagScheduler.handleJobSubmitted(jobId, rdd, func, partitions, callSite, listener, properties)
    // 处理Map Stage提交事件
    case MapStageSubmitted(jobId, dependency, callSite, listener, properties) =>
    dagScheduler.handleMapStageSubmitted(jobId, dependency, callSite, listener, properties)
    // 处理Stage取消事件
    case StageCancelled(stageId) =>
    dagScheduler.handleStageCancellation(stageId)
    // 处理Job取消事件
    case JobCancelled(jobId) =>
    dagScheduler.handleJobCancellation(jobId)
    // 处理Job组取消事件
    case JobGroupCancelled(groupId) =>
    dagScheduler.handleJobGroupCancelled(groupId)
    // 处理所有Job取消事件
    case AllJobsCancelled =>
    dagScheduler.doCancelAllJobs()
    // 处理Executor分配事件
    case ExecutorAdded(execId, host) =>
    dagScheduler.handleExecutorAdded(execId, host)
    // 处理Executor丢失事件
    case ExecutorLost(execId) =>
    dagScheduler.handleExecutorLost(execId, fetchFailed = false)

    case BeginEvent(task, taskInfo) =>
    dagScheduler.handleBeginEvent(task, taskInfo)

    case GettingResultEvent(taskInfo) =>
    dagScheduler.handleGetTaskResult(taskInfo)
    // 处理完成事件
    case completion @ CompletionEvent(task, reason, _, _, taskInfo, taskMetrics) =>
    dagScheduler.handleTaskCompletion(completion)
    // 处理task集失败事件
    case TaskSetFailed(taskSet, reason, exception) =>
    dagScheduler.handleTaskSetFailed(taskSet, reason, exception)
    // 处理重新提交失败Stage事件
    case ResubmitFailedStages =>
    dagScheduler.resubmitFailedStages()
}
```

当Job提交后，JobSubmitted事件会被eventProcessLoop捕获到，然后进入本方法中。开始处理Job，handleJobSubmitted方法执行Stage的划分。

下面详解作业的划分逻辑。

### 2.4 Stage划分

**stage:**

stage(阶段)本身是一个taskset(task集合),各个task的计算函数都是一样的,只是作用于RDD不同的partition(分区)

stage之间以shuffle作为边界,必须等待前面的stage计算完成才能获取数据

**stage dependency(依赖关系)**:

父RDD与子RDD之间存在依赖关系,有两种依赖关系:NarrowDependency(窄依赖)和ShuffleDependency(宽依赖),DAGScheduler根据宽依赖将job划分不同的Stage

子RDD的每个partition只依赖父RDD特定的partition,为窄依赖子RDD的每个partition通常依赖父RDD整个patitions,为宽依赖,需要父RDD操作完成才能开始子RDD的操作

**Stage子类**

stage有两个子类ResultStage`和`ShuffleMapStage。

ShuffleMapStage：

- 属于中间阶段,作为其他stage的输入
- 以Shuffle操作为边界，即依赖为ShuffleDependency(宽依赖)的RDD之前必然是另一个ShuffleMapStage
- stage内部的转换操作(map.filte等)会组成管线,连在一起计算
- 会保存map输出文件

ResultStage：

- 计算Spark的action操作，产生一个结果返回给driver程序或者写到存储种
- 一个job只有一个ResultStage，且一定是最后一个stage

**ActiveJob**

在DAGSchedule中job对应的类为ActiveJob,有两种类型的job：

- resultJob:finalStage类型为ResultStage,执行一个action
- mapstageJob:finalStage类型为ShufMapStage,计算它的map output
	

#### 2.4.1 DAGScheduler#handleJobSubmitted

Stage的划分是从最后一个Stage开始逆推的，每遇到一个宽依赖处，就分裂成另外一个Stage，依此类推直到Stage划分完毕为止。并且，只有最后一个Stage的类型是ResultStage类型。

```scala
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
        // Stage划分过程是从最后一个Stage开始往前执行的，最后一个Stage的类型是ResultStage
        finalStage = createResultStage(finalRDD, func, partitions, jobId, callSite)
    } catch {
        case e: BarrierJobSlotsNumberCheckFailed =>
        logWarning(s"The job $jobId requires to run a barrier stage that requires more slots " +
                   "than the total number of slots in the cluster currently.")
        // If jobId doesn't exist in the map, Scala coverts its value null to 0: Int automatically.
        val numCheckFailures = barrierJobIdToNumTasksCheckFailures.compute(jobId,
                                                                           new BiFunction[Int, Int, Int] {
                                                                               override def apply(key: Int, value: Int): Int = value + 1
                                                                           })
        if (numCheckFailures <= maxFailureNumTasksCheck) {
            messageScheduler.schedule(
                new Runnable {
                    override def run(): Unit = eventProcessLoop.post(JobSubmitted(jobId, finalRDD, func,
                                                                                  partitions, callSite, listener, properties))
                },
                timeIntervalNumTasksCheck,
                TimeUnit.SECONDS
            )
            return
        } else {
            // Job failed, clear internal data.
            barrierJobIdToNumTasksCheckFailures.remove(jobId)
            listener.jobFailed(e)
            return
        }

        case e: Exception =>
        logWarning("Creating new stage failed due to exception - job: " + jobId, e)
        listener.jobFailed(e)
        return
    }
    // Job submitted, clear internal data.
    barrierJobIdToNumTasksCheckFailures.remove(jobId)

    //为该Job生成一个ActiveJob对象，并准备计算这个finalStage
    val job = new ActiveJob(jobId, finalStage, callSite, listener, properties)
    clearCacheLocs()
    logInfo("Got job %s (%s) with %d output partitions".format(
        job.jobId, callSite.shortForm, partitions.length))
    logInfo("Final stage: " + finalStage + " (" + finalStage.name + ")")
    logInfo("Parents of final stage: " + finalStage.parents)
    logInfo("Missing parents: " + getMissingParentStages(finalStage))

    val jobSubmissionTime = clock.getTimeMillis()
    // 该job进入active状态
    jobIdToActiveJob(jobId) = job
    activeJobs += job
    finalStage.setActiveJob(job)
    val stageIds = jobIdToStageIds(jobId).toArray
    val stageInfos = stageIds.flatMap(id => stageIdToStage.get(id).map(_.latestInfo))
    // 向LiveListenerBus发送Job提交事件
    listenerBus.post(
        SparkListenerJobStart(job.jobId, jobSubmissionTime, stageInfos, properties))
    //提交当前Stage
    submitStage(finalStage)
}
```

#### 2.4.2 DAGScheduler#createResultStage

创建job最后一个stage:`ResultStage`,其构造函数要求获取它依赖的父stage,这是进行stage切分的核心,内部从该stage开始向前回溯,遇到`shuffleDependency`就进行切分,如果是`narrowDependency`就归并到同一个stage。

```scala
/**
   * Create a ResultStage associated with the provided jobId.
   *
   * 创建对应jobId的ResultStage。
   */
private def createResultStage(
    rdd: RDD[_],
    func: (TaskContext, Iterator[_]) => _,
    partitions: Array[Int],
    jobId: Int,
    callSite: CallSite): ResultStage = {
    // 先进行阶段校验
    checkBarrierStageWithDynamicAllocation(rdd)
    // 检查屏障阶段是否需要比当前活动插槽总数更多的插槽（以便能够一起启动屏障阶段的所有任务）
    checkBarrierStageWithNumSlots(rdd)
    checkBarrierStageWithRDDChainPattern(rdd, partitions.toSet.size)
    val parents = getOrCreateParentStages(rdd, jobId) //获取或者创建该job最后一个stage:ResultStage之前的stage
    val id = nextStageId.getAndIncrement() //生成一个stageId
    val stage = new ResultStage(id, rdd, func, partitions, parents, jobId, callSite) // 创建最后一个Stage
    stageIdToStage(id) = stage //记录stageId和stage的对应关系
    updateJobIdStageIdMaps(jobId, stage)
    stage
}
```

#### 2.4.3 DAGScheduler#getOrCreateParentStages

找到该RDD的ShuffleDependency,获取对应的ShuffleMapStage，从当前RDD开始使用DFS回朔并进行标记,直到对应路径上遍历到ShuffleDependency或RDD都遍历完成为止。

```scala
/**
   * Get or create the list of parent stages for a given RDD.  The new Stages will be created with
   * the provided firstJobId.
   *
   * 获取或创建给定RDD的父级列表。 将使用提供的firstJobId创建新的阶段。
   */
private def getOrCreateParentStages(rdd: RDD[_], firstJobId: Int): List[Stage] = {
    getShuffleDependencies(rdd).map { shuffleDep =>
        getOrCreateShuffleMapStage(shuffleDep, firstJobId)
    }.toList
}
```

#### 2.4.4 DAGScheduler#getShuffleDependencies

返回指定RDD的直接父项的shuffle-dependencies。此功能不会返回更远的祖先.例如,如果C对B有一个shuffle-dependency,它对A有一个shuffle-dependency:
$$
A <-- B <-- C
$$
使用rdd C调用此函数只会返回 B <-- C dependency

```scala
/**
   * Returns shuffle dependencies that are immediate parents of the given RDD.
    *
    * 返回给定RDD的直接父级的shuffle依赖项。
   *
   * This function will not return more distant ancestors.  For example, if C has a shuffle
   * dependency on B which has a shuffle dependency on A:
   *
   * A <-- B <-- C
   *
   * calling this function with rdd C will only return the B <-- C dependency.
   *
   * This function is scheduler-visible for the purpose of unit testing.
   */
private[scheduler] def getShuffleDependencies(
    rdd: RDD[_]): HashSet[ShuffleDependency[_, _, _]] = {
    val parents = new HashSet[ShuffleDependency[_, _, _]]
    val visited = new HashSet[RDD[_]]
    val waitingForVisit = new ArrayStack[RDD[_]]
    waitingForVisit.push(rdd)
    while (waitingForVisit.nonEmpty) {
        val toVisit = waitingForVisit.pop()
        if (!visited(toVisit)) {
            visited += toVisit
            toVisit.dependencies.foreach {
                case shuffleDep: ShuffleDependency[_, _, _] =>
                parents += shuffleDep
                case dependency =>
                waitingForVisit.push(dependency.rdd)
            }
        }
    }
    parents
}
```

#### 2.4.5 DAGScheduler#getOrCreateShuffleMapStage

如果shuffleIdToMapStage中存在一个shuffleMapStage,则获取它，如果不存在则创建shuffleMapStage。

```scala
/**
   * Gets a shuffle map stage if one exists in shuffleIdToMapStage. Otherwise, if the
   * shuffle map stage doesn't already exist, this method will create the shuffle map stage in
   * addition to any missing ancestor shuffle map stages.
   */
private def getOrCreateShuffleMapStage(
    shuffleDep: ShuffleDependency[_, _, _],
    firstJobId: Int): ShuffleMapStage = {
    shuffleIdToMapStage.get(shuffleDep.shuffleId) match {
        case Some(stage) =>
        stage

        case None =>
        // Create stages for all missing ancestor shuffle dependencies.
        // 找到还未在shuffleToMapStage中注册的ancestors(祖先)shuffleDependencies
        getMissingAncestorShuffleDependencies(shuffleDep.rdd).foreach { dep =>
            // Even though getMissingAncestorShuffleDependencies only returns shuffle dependencies
            // that were not already in shuffleIdToMapStage, it's possible that by the time we
            // get to a particular dependency in the foreach loop, it's been added to
            // shuffleIdToMapStage by the stage creation process for an earlier dependency. See
            // SPARK-13902 for more information.
           // 再次检查避免重复创建(多个ShuffleDependency又依赖到同一个ShuffleDependency的情况)
            if (!shuffleIdToMapStage.contains(dep.shuffleId)) {
                createShuffleMapStage(dep, firstJobId)
            }
        }
        // Finally, create a stage for the given shuffle dependency.
        createShuffleMapStage(shuffleDep, firstJobId)
    }
}
```

#### 2.4.6  DAGScheduler#createShuffleMapStage

创建一个ShuffleMapStage,生成给定的shuffleDependency的分区，如果先前运行的阶段生成相同的shuffle数据,则此函数将复制仍可从先前的shuffle获得的输出位置,以避免不必要地重新生成数据。

```scala
/**
   * Creates a ShuffleMapStage that generates the given shuffle dependency's partitions. If a
   * previously run stage generated the same shuffle data, this function will copy the output
   * locations that are still available from the previous shuffle to avoid unnecessarily
   * regenerating data.
   */
def createShuffleMapStage(shuffleDep: ShuffleDependency[_, _, _], jobId: Int): ShuffleMapStage = {
    val rdd = shuffleDep.rdd
    checkBarrierStageWithDynamicAllocation(rdd)
    checkBarrierStageWithNumSlots(rdd)
    checkBarrierStageWithRDDChainPattern(rdd, rdd.getNumPartitions)
    // ShuffleMapStage中task的数目与rdd分区数目相同（类似于MapReduce的map数量与）
    val numTasks = rdd.partitions.length
    // 该Stage依赖的父Stage,所以这里需要确保Stage从前往后创建
    val parents = getOrCreateParentStages(rdd, jobId)
    // 获取一个stageID，从0开始
    val id = nextStageId.getAndIncrement()
    val stage = new ShuffleMapStage(
        id, rdd, numTasks, parents, jobId, rdd.creationSite, shuffleDep, mapOutputTracker)

    //记录stageId和stage的对应关系
    stageIdToStage(id) = stage
    //记录shuffleId和stage的对应关系
    shuffleIdToMapStage(shuffleDep.shuffleId) = stage
    //记录JobId和stageID的对应关系(Stage的jobIds字段，DAGSchedule的jobIdToStageIds字段)
    updateJobIdStageIdMaps(jobId, stage)

    if (!mapOutputTracker.containsShuffle(shuffleDep.shuffleId)) {
        // Kind of ugly: need to register RDDs with the cache and map output tracker here
        // since we can't do it in the RDD constructor because # of partitions is unknown
        logInfo(s"Registering RDD ${rdd.id} (${rdd.getCreationSite}) as input to " +
                s"shuffle ${shuffleDep.shuffleId}")
        //注册mapOutputTracker，用来记录createShuffleMapStaged的输出
        mapOutputTracker.registerShuffle(shuffleDep.shuffleId, rdd.partitions.length)
    }
    stage
}
```

### 2.5 提交Stage

### 2.5.1 DAGScheduler#submitStage