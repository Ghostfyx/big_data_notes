# Jobs and Scheduling

## 1. Scheduling

Flink中的执行资源是通过任务槽定义的。每个TaskManager将具有一个或多个任务槽，每个任务槽可运行一个并行任务管道。每个并行任务管道由多个连续任务组成，例如：MapFunction的第n个并行实例以及ReduceFunction的第n个并行实例。注意，Flink经常同时执行连续的任务：对于Streaming程序，无论如何都会发生并行执行，但对于批处理程序，它经常发生。

下图说明了这一点，