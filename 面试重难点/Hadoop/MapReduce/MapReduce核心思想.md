# MapReduce核心思想

## 1. MapReduce计算框架优缺点

**优点：**

- **MapReduce 易于编程** Map任务与Reduce任务基于接口实现，用户可以关注与业务逻辑处理而忽略底层计算逻辑。
- **良好的扩展性**  计算资源不足时，通过增加机器的方式横向扩展
- **高容错性**  MapReduce Framework的高容错性针对于节点(master,slaves)容错性和计算任务(map Task和reduce Task)的高可用和错误处理机制
- **适合PB级以上海量数据的离线处理**

**缺点**

- **无法进行实时计算**  一般MR用于+1天时效的处理，或使用Kafuka对接数据仓库，更新/写入HDFS文件，通过调度任务实现准实时
- **不擅长流式计算**  流式计算的输入数据是动态的，而MapReduce的输入数据集是静态的，不能 动态变化
- **不擅长DAG(有向图)计算** 对于机器学习算法或统计学习算法需要大量的迭代计算，MapReduce作业 的输出结果都会写入到磁盘，进行这类计算时需要大量的磁盘IO导致任务效率低下

## 2. 核心思想

![](../../img/mapreduce_core_concept.jpg)

1）MR分布式的运算程序往往需要分成至少 2 个阶段。

2）第一个阶段的 MapTask 并发实例，完全并行运行，互不相干。

3）第二个阶段的 ReduceTask 并发实例互不相干，但是他们的数据依赖于上一个阶段的所有 MapTask 并发实例的输出。

 4）MapReduce 编程模型只能包含一个 Map 阶段和一个 Reduce 阶段，如果用户的业务逻辑非常复杂，那就只能多个MapReduce程序，串行运行。

## 3. MapReduce进程

一个完整的MapReduce程序在分布式运行时有三类实例进程： 

- MrAppMaster：负责整个程序的过程调度及状态协调
- MapTask：负责Map阶段的整个数据处理流程
- ReduceTask：负责Reduce阶段的整个数据处理流程



