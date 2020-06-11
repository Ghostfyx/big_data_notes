# ShuffleMapStage源码分析

## 1， Java Doc

```java
/**
 * ShuffleMapStages are intermediate stages in the execution DAG that produce data for a shuffle.
 * They occur right before each shuffle operation, and might contain multiple pipelined operations
 * before that (e.g. map and filter). When executed, they save map output files that can later be
 * fetched by reduce tasks. The `shuffleDep` field describes the shuffle each stage is part of,
 * and variables like `outputLocs` and `numAvailableOutputs` track how many map outputs are ready.
 *
 * ShuffleMapStages是执行DAG的中间阶段，可以为shuffle生成数据。它们发生在每个Shuffle操作之前，而且在此之前包含多个
 * 流水线操作(例如：map和filter)。在执行时，它们保存map输出文件，随后可以通过reduce任务获取。“shuffleDep”属性描述了
 * 每个Stage所属的shuffle，“outputLocs”和“numAvailableOutputs”之类的变量跟踪准备好多少map输出。
 *
 * ShuffleMapStages can also be submitted independently as jobs with DAGScheduler.submitMapStage.
 * For such stages, the ActiveJobs that submitted them are tracked in `mapStageJobs`. Note that
 * there can be multiple ActiveJobs trying to compute the same shuffle map stage.
 *
 * ShuffleMapStages也可以作为作业通过DAGScheduler.submitMapStage独立提交。对于每一个阶段，
  * 提交他们的ActiveJobs在`mapStageJobs`中进行跟踪
  * 请注意，可能有多个ActiveJob尝试计算相同的ShuffleMapStage
 */
```

## 2. 源码

```java
private[spark] class ShuffleMapStage(
    id: Int,
    rdd: RDD[_],
    numTasks: Int,
    parents: List[Stage],
    firstJobId: Int,
    callSite: CallSite,
    val shuffleDep: ShuffleDependency[_, _, _],
    mapOutputTrackerMaster: MapOutputTrackerMaster)
  extends Stage(id, rdd, numTasks, parents, firstJobId, callSite) {

  private[this] var _mapStageJobs: List[ActiveJob] = Nil

  /**
   * Partitions that either haven't yet been computed, or that were computed on an executor
   * that has since been lost, so should be re-computed.  This variable is used by the
   * DAGScheduler to determine when a stage has completed. Task successes in both the active
   * attempt for the stage or in earlier attempts for this stage can cause paritition ids to get
   * removed from pendingPartitions. As a result, this variable may be inconsistent with the pending
   * tasks in the TaskSetManager for the active attempt for the stage (the partitions stored here
   * will always be a subset of the partitions that the TaskSetManager thinks are pending).
   *
   * 尚未计算的分区，或已经丢失的执行器上已计算的分区，因此应重新计算。 DAGScheduler使用此变量来确定阶段何时完成。
   */
  val pendingPartitions = new HashSet[Int]

  override def toString: String = "ShuffleMapStage " + id

  /**
   * Returns the list of active jobs,
   * i.e. map-stage jobs that were submitted to execute this stage independently (if any).
   */
  def mapStageJobs: Seq[ActiveJob] = _mapStageJobs

  /** Adds the job to the active job list. */
  def addActiveJob(job: ActiveJob): Unit = {
    _mapStageJobs = job :: _mapStageJobs
  }

  /** Removes the job from the active job list. */
  def removeActiveJob(job: ActiveJob): Unit = {
    _mapStageJobs = _mapStageJobs.filter(_ != job)
  }

  /**
   * Number of partitions that have shuffle outputs.
   * When this reaches [[numPartitions]], this map stage is ready.
    * 具有shuffle map output的分区数
    *
   */
  def numAvailableOutputs: Int = mapOutputTrackerMaster.getNumAvailableOutputs(shuffleDep.shuffleId)

  /**
   * Returns true if the map stage is ready, i.e. all partitions have shuffle outputs.
   */
  def isAvailable: Boolean = numAvailableOutputs == numPartitions

  /** Returns the sequence of partition ids that are missing (i.e. needs to be computed). */
  override def findMissingPartitions(): Seq[Int] = {
    mapOutputTrackerMaster
      .findMissingPartitions(shuffleDep.shuffleId)
      .getOrElse(0 until numPartitions)
  }
}
```

