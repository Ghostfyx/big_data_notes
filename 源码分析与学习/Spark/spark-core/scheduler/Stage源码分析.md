# Stage源码分析

## 1. Java doc

```java
/**
 * A stage is a set of parallel tasks all computing the same function that need to run as part
 * of a Spark job, where all the tasks have the same shuffle dependencies. Each DAG of tasks run
 * by the scheduler is split up into stages at the boundaries where shuffle occurs, and then the
 * DAGScheduler runs these stages in topological order.
 *
 * Stage由一组运行相同的计算的并行task组成，task作为Spark作业的一部分进行，所有任务都具有相同的随机依赖关系。
 * 调度程序运行的每个DAG任务在发生Shuffle的边界处分为多个阶段，然后DAGScheduler以拓扑顺序运行这些阶段。
 *
 * Each Stage can either be a shuffle map stage, in which case its tasks' results are input for
 * other stage(s), or a result stage, in which case its tasks directly compute a Spark action
 * (e.g. count(), save(), etc) by running a function on an RDD. For shuffle map stages, we also
 * track the nodes that each output partition is on.
 *
 * 每个阶段可以是shuffleMap阶段，其任务的结果输入到其他阶段；也可以是ResultStage，在这种情况下直接计算Spark Action操作
 * 例如：count()， save()等。对于shuffle map stages，我们还可以跟踪每个输出分区所在的节点。
 *
 * Each Stage also has a firstJobId, identifying the job that first submitted the stage.  When FIFO
 * scheduling is used, this allows Stages from earlier jobs to be computed first or recovered
 * faster on failure.
 *
 * 每个阶段还具有一个firstJobId，用于标识首先提交该阶段的作业。
 * 使用FIFO调度时，允许首先计算较早作业的阶段，或者在出现故障时更快地恢复。
 *
 * Finally, a single stage can be re-executed in multiple attempts due to fault recovery. In that
 * case, the Stage object will track multiple StageInfo objects to pass to listeners or the web UI.
 * The latest one will be accessible through latestInfo.
 *
 * 最后，由于故障恢复，可以多次尝试重新执行一个阶段。 在这种情况下，
  * Stage对象将跟踪多个StageInfo对象以传递给侦听器或Web UI。 最新的将可以通过latestInfo访问。
 *
 * @param id Unique stage ID 唯一StageID
 * @param rdd RDD that this stage runs on: for a shuffle map stage, it's the RDD we run map tasks
 *   on, while for a result stage, it's the target RDD that we ran an action on
  *
 * 此阶段运行的RDD：对于shuffle map stage，是在其上运行map task的RDD；对于result stage，是对其执行Action操作的目标RDD
  *
 * @param numTasks Total number of tasks in stage; result stages in particular may not need to
 *   compute all partitions, e.g. for first(), lookup(), and take().
 * Stage中的task总数，result stages可能不需要计算所有分区。例如：first()，lookup()，take()
 *
 * @param parents List of stages that this stage depends on (through shuffle dependencies).
 * 当前Stage依赖的一组Stages(通过Shuffle dependencies)
 *
 * @param firstJobId ID of the first job this stage was part of, for FIFO scheduling.
 * 此阶段第一部分作业的ID，用于FIFO调度。
 *
 * @param callSite Location in the user program associated with this stage: either where the target
 *   RDD was created, for a shuffle map stage, or where the action for a result stage was called.
 */
```

## 2. Stage源码

Stage是一个抽象类，其有两个子类：ShuffleMapStage和ResultStage。

```scala
private[scheduler] abstract class Stage(
    val id: Int,
    val rdd: RDD[_],
    val numTasks: Int,
    val parents: List[Stage],
    val firstJobId: Int,
    val callSite: CallSite)
  extends Logging {

  val numPartitions = rdd.partitions.length

  /** Set of jobs that this stage belongs to. */
  val jobIds = new HashSet[Int]

  /** The ID to use for the next new attempt for this stage. */
  private var nextAttemptId: Int = 0

  val name: String = callSite.shortForm
  val details: String = callSite.longForm

  /**
   * Pointer to the [[StageInfo]] object for the most recent attempt. This needs to be initialized
   * here, before any attempts have actually been created, because the DAGScheduler uses this
   * StageInfo to tell SparkListeners when a job starts (which happens before any stage attempts
   * have been created).
    *
    * 指向最近尝试的[[StageInfo]]对象的指针。 在实际创建任何尝试之前，
    * 需要在此进行初始化，因为DAGScheduler使用此StageInfo告诉SparkListeners作业何时开始
    * （发生在创建任何阶段尝试之前）
   */
  private var _latestInfo: StageInfo = StageInfo.fromStage(this, nextAttemptId)

  /**
   * Set of stage attempt IDs that have failed. We keep track of these failures in order to avoid
   * endless retries if a stage keeps failing.
   * We keep track of each attempt ID that has failed to avoid recording duplicate failures if
   * multiple tasks from the same stage attempt fail (SPARK-5945).
   */
  val failedAttemptIds = new HashSet[Int]

  private[scheduler] def clearFailures() : Unit = {
    failedAttemptIds.clear()
  }

  /** Creates a new attempt for this stage by creating a new StageInfo with a new attempt ID. */
  def makeNewStageAttempt(
      numPartitionsToCompute: Int,
      taskLocalityPreferences: Seq[Seq[TaskLocation]] = Seq.empty): Unit = {
    val metrics = new TaskMetrics
    metrics.register(rdd.sparkContext)
    _latestInfo = StageInfo.fromStage(
      this, nextAttemptId, Some(numPartitionsToCompute), metrics, taskLocalityPreferences)
    nextAttemptId += 1
  }

  /** Returns the StageInfo for the most recent attempt for this stage. */
  def latestInfo: StageInfo = _latestInfo

  override final def hashCode(): Int = id

  override final def equals(other: Any): Boolean = other match {
    case stage: Stage => stage != null && stage.id == id
    case _ => false
  }

  /** Returns the sequence of partition ids that are missing (i.e. needs to be computed). */
  def findMissingPartitions(): Seq[Int]
}
```

