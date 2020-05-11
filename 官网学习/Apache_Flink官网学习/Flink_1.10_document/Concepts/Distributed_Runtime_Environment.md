# Distributed Runtime Environment

## 1. Tasks and Operator Chains

对于分布式执行，Flink将sub task链接组成task，每个任务由一个线程执行。将操作符链接一起组成任务是非常有用的优化：它减少了线程间的切换和缓冲的开销。在总体增加吞吐量的同时，减少延迟。可以配置链接行为，详情可见[chaining docs](https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/stream/operators/#task-chaining-and-resource-groups)。

下图中的示例数据流是由五个子任务执行的，因此有五个并行线程。

![](../../img/tasks_chains.svg)

## 2. Job Managers, Task Managers, Clients

Flink运行时包含两类进程：

- **JobManagers**(也称作master)协调分布式任务执行，它们安排任务，协调检查点，协调故障恢复等。集群中至少有一个Job Manager。高可用性设置将有多个JobManager，其中一个始终是领导者，而其他则是备用的。
- **TaskManagers** (也称为worker)执行dataflow的任务(准确的说是sub task)，并缓冲和交换数据流。必须至少有一个TaskManager。

JobManager和TaskManager能以不同方式启动：

- 直接作为独立集群运行，即standalone方式；
- 在容器中或由资源管理框架(YARN、Mesos)管理；

 TaskManager连接到JobManager，宣布自己可用，并被分配工作。

client不是运行和程序执行时的一部分，被用来准备和发送dataflow到jobManager，之后，客户端可以断开连接或保持连接状态以接收进度报告。客户端可以作为Java/Scala程序的一部分执行，也可以在命令行./bin/flink run ...中运行。

![](../../img/processes.svg)

## 3. Task Slots and Resources

每一个worker(TaskManager)都是一个JVM进程，并且可以在单独的线程中执行一个或多个子任务。每个worker接收的任务数取决于worker中task slots的数量。PS：每个worker至少有一个task slots。

每个task slot代表TaskManager的固定资源子集。例如，具有三个task slot的TaskManager会将其托管内存的1/3专用于每个slot。分配资源意味着子任务不会与其他作业的子任务竞争托管内存，而是具有一定数量的保留托管内存。请注意：**没有将CPU隔离**。 当前slot仅将任务的托管内存分开。

通过调整task slot的数量，用户可以定义子任务如何相互隔离。对于TaskManager，具有一个slot，意味着每个任务组都在单独的JVM中运行(例如，可以在单独的容器中启动)；具有多个slot意味着多个sub task共享同一JVM，同一JVM中的任务共享TCP连接(通过多路复用)和心跳消息。 还可以共享数据集和数据结构，从而减少每个任务的开销。

![](../../img/tasks_slots.svg)

默认情况下，Flink允许不同任务中的子任务共享同一slot，只要他们是同一Job即可。结果是一个slot可以容纳整个作业管道。 允许slot共享有两个主要好处：

- Flink集群所需的task slot与作业中的最高并行度恰好一样多。 无需计算一个程序总共包含多少个任务(并行度各不相同)
- 更容易获得更好的资源利用率。 如果没有slot共享，则非密集型source/map()子任务将阻塞与资源密集型窗口子任务一样多的资源。 通过共享slot，示例中的基本并行度从2增加到6，可以充分利用插槽资源，同时确保繁重的子任务在TaskManager之间公平分配。

![](../../img/slot_sharing.svg)

API还包括*[资源组](https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/stream/operators/#task-chaining-and-resource-groups)*机制，可用于防止不良的slot共享。

根据经验，较好的默认任务slot数量应该是CPU内核的数量。 使用超线程，每个插槽将占用2个或更多硬件线程上下文。

## 4. State Backends

存储键/值索引的数据结构取决于所选的状态后端。一个状态后端将数据存储在内存中的HashMap中，另一个状态后端使用RocksDB作为键/值存储。除了定义保存状态的数据结构之外，状态后端还实现获取键/值状态的时间点快照并将该快照存储为检查点一部分。

![](../../img/checkpoints.svg)

## 5. SavePoints

用DataStream API编写的程序可以从保存点恢复执行。 保存点允许更新程序和Flink群集，而不会丢失任何状态。

[Savepoints](https://ci.apache.org/projects/flink/flink-docs-release-1.10/ops/state/savepoints.html)是手动触发的检查点，为程序创建快照并将其写到状态后端，为此依靠常规的检查点机制。在执行期间，程序会定期在工作节点上快照并产生检查点。 为了进行恢复，仅需要最后完成的检查点，并且可以在新的检查点完成后立即安全地丢弃较旧的检查点。

savePoint与这些定期checkPoint类似，除了它们由用户触发并且在更新的检查点完成时不会自动过期。 可以从命令行或通过REST API取消作业时创建savePoint。