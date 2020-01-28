# Presto实现原理

Facebook的数据仓库存储在少量大型Hadoop/HDFS集群。Hive是Facebook在几年前专为Hadoop打造的一款数据仓库工具。在以前，Facebook的科学家和分析师一直依靠Hive来做数据分析。但Hive使用MapReduce作为底层计算框架，是专为批处理设计的。但随着数据越来越多，使用Hive进行一个简单的数据查询可能要花费几分到几小时，显然不能满足交互式查询的需求。Facebook也调研了其他比Hive更快的工具，但它们要么在功能有所限制要么就太简单，以至于无法操作Facebook庞大的数据仓库。

2012年开始试用的一些外部项目都不合适，他们决定自己开发，这就是Presto。2012年秋季开始开发，目前该项目已经在超过 1000名Facebook雇员中使用，运行超过30000个查询，每日数据在1PB级别。Facebook称Presto的性能比Hive要好上10倍多。2013年Facebook正式宣布开源Presto。

本文首先介绍Presto从用户提交SQL到执行的这一个过程，然后尝试对Presto实现实时查询的原理进行分析和总结

## 1. Presto 结构

![img](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2014/2c911d9d.png)

Presto查询引擎是一个Master-Slave的架构，由一个Coordinator节点，一个Discovery Server节点，多个Worker节点组成，Discovery Server通常内嵌于Coordinator节点中。Coordinator负责解析SQL语句，生成执行计划，分发执行任务给Worker节点执行。Worker节点负责实际执行查询任务。Worker节点启动后向Discovery Server服务注册，Coordinator从Discovery Server获得可以正常工作的Worker节点。如果配置了Hive Connector，需要配置一个Hive MetaStore服务为Presto提供Hive元信息，Worker节点与HDFS交互读取数据。2. 

## 2. Presto执行查询过程简介

presto实现低延时查询的原理关键在以下几点：

1. 完全基于内存的并行计算
2. 流水线
3. 本地化计算
4. 动态编译执行计划
5. 小心使用内存和数据结构
6. 类BlinkDB的近似查询
7. GC控制

为了介绍上述几个要点，这里先介绍一下Presto执行查询的过程。

### 2.1 提交查询

用户使用Presto Cli提交一个查询语句后，Cli使用HTTP协议与Coordinator通信，Coordinator收到查询请求后调用SqlParser解析SQL语句得到Statement对象，并将Statement封装成一个QueryStarter对象放入线程池中等待执行。

![img](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2014/cc67b533.jpg)

### 2.2 SQL编译过程

Presto与Hive一样，使用Antlr编写SQL语法，语法规则定义在Statement.g和StatementBuilder.g两个文件中。 如下图中所示从SQL编译为最终的物理执行计划大概分为5部，最终生成在每个Worker节点上运行的LocalExecutionPlan，这里不详细介绍SQL解析为逻辑执行计划的过程，通过一个SQL语句来理解查询计划生成之后的计算过程。

![img](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2014/e268eb9c.png)

![img](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2014/edf9a1bb.jpg)

### 2.3 物理执行计划

逻辑执行计划图中的虚线就是Presto对逻辑执行计划的切分点，逻辑计划Plan生成的SubPlan分为四个部分，每一个SubPlan都会提交到一个或者多个Worker节点上执行。

SubPlan有几个重要的属性planDistribution、outputPartitioning、partitionBy属性。

![img](https://awps-assets.meituan.net/mit-x/blog-images-bundle-2014/029b5945.png)

1. PlanDistribution表示一个查询Stage的分发方式，逻辑执行计划图中的4个SubPlan共有3种不同的PlanDistribution方式：Source表示这个SubPlan是数据源，Source类型的任务会按照数据源大小确定分配多少个节点进行执行；Fixed表示这个SubPlan会分配固定的节点数进行执行（Config配置中的query.initial-hash-partitions参数配置，默认是8）；None表示这个SubPlan只分配到一个节点进行执行。在下面的执行计划中，SubPlan1和SubPlan0 PlanDistribution=Source，这两个SubPlan都是提供数据源的节点，SubPlan1所有节点的读取数据都会发向SubPlan0的每一个节点；SubPlan2分配8个节点执行最终的聚合操作；SubPlan3只负责输出最后计算完成的数据。
2. OutputPartitioning属性只有两个值HASH和NONE，表示这个SubPlan的输出是否按照partitionBy的key值对数据进行Shuffle。在下面的执行计划中只有SubPlan0的OutputPartitioning=HASH，所以SubPlan2接收到的数据是按照rank字段Partition后的数据。