# Dataflow Programming Model

## 1. API抽象层次

Flink提供了不同级别的抽象来开发流/批处理应用程序。

![](../../img/levels_of_abstraction.svg)

- 最低级别的抽象仅提供状态流。 它通过Process Function嵌入到DataStream API中。 它允许用户自由地处理一个或多个流中的事件，并使用一致的容错状态。 此外，用户可以注册事件时间和处理时间回调，从而允许程序实现复杂的计算。

- 实际上，大多数应用程序不需要上述低级抽象，而是针对核心API进行编程，例如DataStream API(有界/无界流)和DataSet API(有界数据集)。 这些API提供了用于数据处理的通用构件，例如各种形式的用户指定的转换，联接，聚合，窗口，状态等。这些API中处理的数据类型以相应编程语言中的类表示。

	低级Process Function与DataStream API集成在一起，从而使得仅对某些操作进行低级抽象成为可能。 DataSet API在有限的数据集上提供了其他原语，例如循环/迭代。

- Table API是围绕表的声明性DSL，它可以动态更改表(在表示流时)。Table API遵循(扩展的)关系模型：表具有附加Schema(类似于关系型数据库中的表)。并且该API提供可比较的操作，例如：select, project, join, group-by, aggregate。Table API程序 以声明方式定义应执行的逻辑操作，而不是确切指定操作代码的外观。 尽管Table API可以由各种类型的用户定义函数扩展，但它的表达性不如Core API，但使用起来更为简洁（无需编写代码）。 此外，Table API程序还经过优化程序，该优化程序在执行之前应用优化规则。

	可以在表和DataStream / DataSet之间无缝转换，从而允许程序将Table API以及DataStream和DataSet API混合使用。

- Flink提供的最高级别的抽象是SQL。 这种抽象在语义和表达方式上均类似于Table API，但是将程序表示为SQL查询表达式。 SQL抽象与Table API紧密交互，并且可以对Table API中定义的表执行SQL查询。

## 2. Programs and Dataflows

Flink程序的基本构建块是流和转换(Flink的DataSet API中使用的DataSet也在内部是流-稍后将进行更多说明)。从概念上讲，流是数据记录流(可能永无止境)，转换是将一个或多个流作为输入，并产生一个或多个输出流的操作。

执行时，Flink程序将映射到流和转换运算符组成的流数据流。每个数据流都以一个或多个源开始，并以一个或多个接收器结束。数据流类似于任意有向无环图(DAG)。尽管可以通过迭代构造来允许特殊形式的循环，但是大多数情况下，为简单起见，将忽略这一点。

![](../../img/program_dataflow.svg)

程序中的转换与数据流中的运算符之间通常存在一一对应的关系。 但是，有时，一个转换可能包含多个转换运算符。

source和sinks在 [streaming connectors](https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/connectors/index.html)和[batch connectors](https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/batch/connectors.html)文档中。转换操作在文档[DataStream operators](https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/stream/operators/index.html)和[DataSet transformations](https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/batch/dataset_transformations.html)中。

## 3. Dataflows并行度

Flink中的程序本质上是并行的和分布式的。 在执行期间，一个流具有一个或多个流分区，每个操作具有一个或多个操作子任务。 操作子任务彼此独立，并在不同的线程中执行，并且可能在不同的机器或容器上执行。

操作子任务的数量是该操作的并行度。数据流的并行性始终是其操作的并行度。同一程序的不同运算符可能具有不同的并行度。

![](../../img/parallel_dataflow.svg)

流可以按一对一(或转发)模式或重新分配模式在两个运算符之间传输数据：

- **One-to-one streams**：例如，上图中的Source和map( )运算符之间，保留元素的分区和排序。 这意味着map()运算符的subtask [1]将看到与Source运算符的subtask [1]产生的元素相同的顺序。
- **Redistributing Stream**：如以上的map( )和keyBy/window之间，以及keyBy/window和Sink之间，更改流的分区。每个操作子任务都将数据发送到不同的目标子任务，具体取决于所选的转换。  例如：keyBy() 通过散列键重新分区，broadcast()或rebalance()随机重新分区。在重新分配交换中，元素之间的顺序仅保留在每对发送和接收子任务中，例如，map()的subtask [1]和keyBy/window的subtask[2]。 因此，在此示例中，保留了每个键内的顺序，但是并行性的确引入了不确定性，即不同键的聚合结果到达接收器的顺序。

配置和控制并行度参见文档：[parallel execution](https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/parallel.html)。

## 4. Windows

聚合事件(例如：counts, sums)在流上的工作方式与批处理不同。 例如，不可能计算流中的所有元素，因为流通常是无限的。 流的聚合(counts, sum等)由窗口确定范围，例如最近5分钟的计数或最近100个元素的总和。

window可以作为时间事件(例如：每30秒)或数据驱动的(例如：每100个元素)。通常可以区分不同类型的窗口，例如滚动窗口(无重叠)，滑动窗口(有重叠)和会话窗口(由不活动的间隙打断)。

![](../../img/windows.svg)

## 5. Time

在流式传输程序中引用时间时(例如，定义窗口)，可以引用不同的时间概念：

- **Event Time**：创建事件的时间。 通常用事件中的时间戳记来描述，例如由生产传感器或生产服务附加。 Flink通过时间戳分配器访问事件时间戳
- **Ingestion time**：事件在数据源处进入Flink数据流的时间
- **Processing Time**：每个执行基于时间的操作的本地时间

![](../../img/event_ingestion_processing_time.svg)

更多关于操作时间见文档：[event time docs](https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/event_time.html)

## 6. Stateful Operations

尽管数据流中的许多操作一次仅查看一个事件(例如：事件解析器)。但某些操作会记住多个事件的信息(例如窗口运算符)，这些操作称为有状态。

有状态操作的状态以可以被认为是嵌入式键/值存储的方式维护。 严格将状态与有状态运算符读取的流一起进行分区和分发。 因此，只有在keyBy()函数之后，才可以在键控流上访问键/值状态，对齐流键和状态确保所有状态更新都是本地操作，从而确保一致性而没有事务开销。 这种对齐方式还允许Flink重新分配状态并透明地调整流分区。

![](../../img/state_partitioning.svg)

## 7. Checkpoints for Fault Tolerance

Flink通过流重播和检查点来实现容错。检查点与每个输入流中的特定点以及每个运算符的对应状态有关。通过恢复操作的状态并从检查点开始重放事件，可以在保持一致性(完全一次处理语义)的同时从检查点恢复流式数据流。

检查点间隔是在执行过程中权衡容错开销与恢复时间(需要重播的事件数)的一种方法。

[fault tolerance internals](https://ci.apache.org/projects/flink/flink-docs-release-1.10/internals/stream_checkpointing.html)提供了有关Flink如何管理检查点和相关主题的更多信息。 有关启用和配置检查点的详细信息，请参见[checkpointing API docs](https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/stream/state/checkpointing.html)。

## 8. Batch on **Streaming**

Flink执行批处理程序作为流程序的特例，在这种情况下，流是有界的(元素数量有限)。 DataSet在内部被视为数据流。 因此，以上概念以同样的方式适用于批处理程序，也适用于流式程序，但有少量例外：

- [Fault tolerance for batch programs](https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/batch/fault_tolerance.html)不使用checkPoint机制，通过完全重播流来进行恢复。 这是可能的，因为输入是有界的。 这将成本更多地倾向于恢复，但由于避免了检查点，因此使常规处理的成本降低了。
- DataSet API中的状态操作使用简化的内存中/内核外数据结构，而不是键/值索引。
- DataSet API引入了特殊的同步(基于超步)迭代，仅在有限流上才有可能。更多信息详见[iteration docs](https://ci.apache.org/projects/flink/flink-docs-release-1.10/dev/batch/iterations.html)。