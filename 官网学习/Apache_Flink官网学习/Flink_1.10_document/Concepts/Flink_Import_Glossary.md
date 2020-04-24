# Glossary

## 1. Flink Application Cluster

Flink应用集群是专用的Flink集群，仅执行一个Flink作业。Flink群集的生存期与Flink作业的生存期绑定在一起。 在工作模式下，以前的Flink应用程序集群也称为Flink集群。

## 2. Flink Cluster

一种分布式系统，通常由一个Flink Master和一个或多个Flink TaskManager进程组成。

## 3. Event

一个事件是由应用程序建模的域状态的更改。事件可以是流或批处理应用程序的输入和/或输出。 事件是特殊类型的记录。

## 4. ExecutionGraph

参考[Physical Graph](https://ci.apache.org/projects/flink/flink-docs-release-1.10/concepts/glossary.html#physical-graph)

## 5. Function

方法由用户实现，并封装Flink程序的应用程序逻辑。 大多数方法由相应的运算符包装。

## 6. Instance

实例用于描述运行时特定类型(通常是运算符或函数)的特定实例。由于Apache Flink主要是由Java编写， 因此它对应于Java中的Instance或Object的定义。在Apache Flink上下文中，术语“parallel instance”经常被用来强调相同操作符或函数类型的多个实例正在并行运行。

## 7. Flink Job

Flink job是Flink项目运行时的表示。Flink作业既可以提交到长时间运行的Flink会话群集，也可以作为独立的Flink应用程序群集启动。

## 8. JobGraph

参考[Logical Graph](https://ci.apache.org/projects/flink/flink-docs-release-1.10/concepts/glossary.html#logical-graph)

## 9. Flink JobManager

JobManager是运行在Flink Master中的一个组件。JobManger负责监视单个作业中任务的执行情况。历史版本中，整个Flink Master都称为JobManager。

## 10. Logical Graph

逻辑图是描述流处理程序的高级逻辑的有向图。 节点是运算符，边表示运算符的输入/输出关系，并且对应于数据流或数据集。

## 11. Managed State

状态管理描述了已在框架中注册的应用程序状态。Flink负责维护其持久性与扩展性。

## 12. Flink Master

Flink Master是Flink集群中的master角色。它包含三种不同组件：Flink Resource Manager, Flink Dispatcher和每个Flink Job的Flink JobManager。

## 13. Operator

Logical Graph中的一个节点。Operator执行某种操作，通常由Function执行。source和sink是用于数据摄取和数据出口的特殊运算符。

## 14. Operator Chain

一个操作链包含两个或多个连续的操作组成，操作之间没有任何重新分区。同一操作链中的操作之间转发记录无需进行序列化或Flink网络堆栈。

## 15. Partition

分区是整个数据流或数据集的独立子集。 通过将每个记录分配给一个或多个分区，将数据流或数据集划分为多个分区。 任务在运行时使用数据流或数据集的分区。 改变数据流或数据集分区方式的转换通常称为重新分区。

## 16. Physical Graph

物理图是在分布式运行中逻辑图转换的结果。 节点是任务，边表示数据流或数据集的输入/输出关系或分区。

## 17. Record

记录是数据集或数据流的组成元素。 运算符和函数接收记录作为输入，并发出记录作为输出。

## 18. Flink Session Cluster

长期运行的Flink群集，可以接受多个Flink作业执行。Flink群集的生命周期未绑定到任何Flink作业的生存期。以前，Flink会话群集在会话模式下也称为Flink群集。

## 19. State Backend

对于流处理程序，Flink作业的状态后端确定如何在每个TaskManager(TaskManager的Java堆或嵌入式RocksDB)上存储其状态，以及在检查点上写入状态的位置(Flink Master的JVM堆或文件系统)

## 20. Sub-Task

子任务是负责处理流数据各个分区内记录的任务。Sub-Task强调统一操作或操作链具有多个并行任务。

## 21. Task

物理图中的节点，task是基本工作单元，由Flink的运行时执行。 任务恰好封装了一个操作或操作链的并行实例。

## 22. Flink TaskManager

TaskManager是Flink集群的工作进程，任务被调度在TaskManager中执行。TaskManger彼此通信以在后续任务之间交换数据。

## 23. Transformation

将转换应用于一个或多个数据流或数据集，并产生一个或多个输出数据流或数据集。转换可能会更改每个记录的数据流或数据集，但也可能仅更改其分区或执行聚合。尽管操作和函数是Flink API的“物理”部分，但转换只是API概念。具体来说，大多数转换是由操作执行的。