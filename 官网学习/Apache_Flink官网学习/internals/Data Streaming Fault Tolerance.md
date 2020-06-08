# Data Streaming Fault Tolerance

## 1. Introduction

Apache Flink提供了一种容错机制，可以一致地恢复数据流应用程序的状态。该机制可确保即使出现故障，程序的状态最终也将恰好一次反映出数据流中的每条记录。 请注意，有一个开关可以降级为at least once。

容错机制连续绘制分布式流数据的快照。对于状态小的流应用程序，这些快照非常轻巧，可以在不影响性能的情况下频繁绘制。 流应用程序的状态存储在可配置的位置(例如Master节点或HDFS)。

如果发生程序故障(由于机器，网络或软件故障)，Flink将停止分布式流数据流。随后，系统重新启动operators，并将其重新设置为最新一次的成功检查点。 输入流将重置为状态快照的点。确保作为重新启动的并行数据流的一部分处理的任何记录都不属于先前的检查点状态。

注意：默认情况下，检查点是禁用的。 有关如何启用和配置检查点的详细信息，请参见[Checkpointing](https://ci.apache.org/projects/flink/flink-docs-release-1.10/zh/dev/stream/state/checkpointing.html)。

注意：为了使该机制实现其全部保证，数据流源(例如消息队列或broker)必须能够将流后退到定义的最近点。