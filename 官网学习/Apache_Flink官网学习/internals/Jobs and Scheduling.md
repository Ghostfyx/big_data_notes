# Jobs and Scheduling

## 1. Scheduling

Flink中的执行资源是通过任务槽定义的。每个TaskManager将具有一个或多个任务槽，每个任务槽可运行一个并行任务管道。每个并行任务管道由多个连续任务组成，例如：MapFunction的第n个并行实例以及ReduceFunction的第n个并行实例。注意，Flink经常同时执行连续的任务：对于Streaming程序，无论如何都会发生并行执行，但对于批处理程序，它经常发生。

下图说明了这一点，一个具有data source，MapFunction和ReduceFunction的程序，Source和MapFunction以4的并行度执行，而ReduceFunction以3的并行度执行。管道由Source - Map - Reduce序列构成。在具有3个slot的2个TaskManager集群上，将按以下说明执行程序。

<img src="https://ci.apache.org/projects/flink/flink-docs-release-1.10/fig/slots.svg" style="zoom:67%;" />

在内部，Flink通过SlotSharingGroup和CoLocationGroup定义哪些任务可以共享一个插槽(允许)，哪些任务必须严格放置在同一插槽中。

## 2. JobManager Data Structures

在作业执行期间，JobManager会跟踪分布式任务，决定何时安排下一个(一组)任务，并对完成的任务或执行失败的任务作出反应。

JobManager接收[JobGraph](https://github.com/apache/flink/blob/master//flink-runtime/src/main/java/org/apache/flink/runtime/jobgraph/)，它是由操作算子([JobVertex](https://github.com/apache/flink/blob/master//flink-runtime/src/main/java/org/apache/flink/runtime/jobgraph/JobVertex.java))和中间结果([IntermediateDataSet](https://github.com/apache/flink/blob/master//flink-runtime/src/main/java/org/apache/flink/runtime/jobgraph/IntermediateDataSet.java))组成的数据流的表示。每个运算符都有属性，例如并行性和它执行的代码。 此外，JobGraph具有一组附加的库，这些库对于执行操作算子的代码是必需的。

JobManager将JobGraph转换为一个[ExecutionGraph](https://github.com/apache/flink/blob/master//flink-runtime/src/main/java/org/apache/flink/runtime/executiongraph/)。ExecutionGraph是JobGraph的并行版本：对于每一个JobVertex，每一个并行子任务都包含一个ExecutionVertex。并行度为100的操作算子将具有一个JobVertex和100个ExecutionVertices。

ExecutionVertex跟踪特定子任务的执行状态。 一个JobVertex中的所有ExecutionVertices都保存在ExecutionJobVertex中，该ExecutionJobVertex可以跟踪操作员的整体状况。 除顶点外，ExecutionGraph还包含IntermediateResult和IntermediateResultPartition。 前者跟踪IntermediateDataSet的状态，后者跟踪其每个分区的状态。

<img src="https://ci.apache.org/projects/flink/flink-docs-release-1.10/fig/job_and_execution_graph.svg" style="zoom:80%;" />

每个ExecutionGraph都有与之关联的作业状态。 该作业状态指示作业执行的当前状态。

Flink作业首先处于创建状态，然后切换到运行状态，并在完成所有工作后切换到完成状态。发生故障时，作业会首先切换到失败，然后取消正在运行的任务。如果所有作业顶点都已达到最终状态并且该作业不可重新启动，则该作业将转换为失败。 如果作业可以重新启动，则它将进入重新启动状态。 作业完全重新启动后，它将达到创建状态。

如果用户取消作业，它将进入取消状态。 这也需要取消所有当前正在运行的任务。 一旦所有正在运行的任务都达到了最终状态，作业便转换为已取消状态。

与完成状态，取消状态和失败状态不同，它们表示全局终端状态，因此触发清理作业，而挂起状态仅是本地终端。 在本地终端意味着作业的执行已在相应的JobManager上终止，但是Flink群集的另一个JobManager可以从持久性HA存储中检索该作业并重新启动它。 因此，达到暂停状态的作业将无法完全清理。

<img src="https://ci.apache.org/projects/flink/flink-docs-release-1.10/fig/job_status.svg" style="zoom:70%;" />

在执行ExecutionGraph的过程中，每个并行任务都经历了多个阶段，从创建到完成或失败。 下图说明了状态以及状态之间的可能转换。 一个任务可以执行多次(例如，在故障恢复过程中)。 因此，在Execution中跟踪ExecutionVertex的执行。 每个ExecutionVertex都有一个当前的Execution和先前的Execution。

![](https://ci.apache.org/projects/flink/flink-docs-release-1.10/fig/state_machine.svg)