# Apache Kylin VS Apache Doris

Apache Kylin 和 Apache Doris 都是优秀的开源OLAP系统，本文将全方位地对比Kylin和Doris。Kylin和Doris分别是MOALP和ROLAP的代表，对比这两个系统的目的不是为了说明哪个系统更好，**只是为了明确每个系统的设计思想和架构原理，让大家可以根据自己的实际需求去选择合适的系统，也可以进一步去思考我们如何去设计出更优秀的OLAP系统**。

## 1. 系统架构

### 1.1 Kylin

Kylin的核心思想是**预计算**，**利用空间换时间来加速查询模式固定的OLAP查询**。

Kylin的理论基础是Cube理论，每一种维度组合称之为Cuboid，所有Cuboid的集合是Cube。 其中由所有维度组成的Cuboid称为Base Cuboid，图中(A,B,C,D)即为Base Cuboid，所有的Cuboid都可以基于Base Cuboid计算出来。 在查询时，Kylin会自动选择满足条件的最“小”Cuboid，比如下面的SQL就会对应Cuboid（A,B）:

> select xx from table where A=xx group by B

![](../img/kylin-architecture.jpg)

下图是Kylin数据流转的示意图，Kylin自身的组件只有两个：**JobServer和QueryServer**。Kylin的JobServer主要负责将数据源(Hive，Kafka)的数据通过计算引擎(MapReduce，Spark)生成Cube存储到存储引擎(HBase)中；QueryServer负责SQL解析，逻辑计划的生成和优化，向HBase的多个Region发起请求，并对多个Region的结果进行汇总，生成最终的结果集。

![kylin-dataflow.jpg](../img/kylin-dataflow.jpg)

下图是Kylin可插拔的架构图, 在架构设计上，Kylin的**数据源**，构建Cube的**计算引擎**，**存储引擎**都是可插拔的。Kylin的核心就是这套可插拔架构，Cube数据模型和Cuboid的算法。

![](../img/kylin-pluggable-architecture.jpg)

## 1.2 Doris

Doris是一个MPP的OLAP系统，主要整合了**Google Mesa**（数据模型），**Apache Impala**（MPP Query Engine)和**Apache ORCFile** (存储格式，编码和压缩) 的技术。

![](../img/Doris-theory.jpg)

Doris的系统架构如下，Doris主要分为FE和BE两个组件，FE主要负责查询的编译，分发和元数据管理(基于内存，类似HDFS NN)；BE主要负责查询的执行和存储系统。

![](../img/Doris-architecher.jpg)

## 2. 数据模型

### 2.1 Kylin数据模型

Kylin将表中的列分为维度列和指标列。在数据导入和查询时相同维度列中的指标会按照对应的聚合函数(Sum, Count, Min, Max, 精确去重，近似去重，百分位数，TOPN)进行聚合。

在存储到HBase时，**Cuboid+维度 会作为HBase的Rowkey, 指标会作为HBase的Value**，一般所有指标会在HBase的一个列族，每列对应一个指标，但对于较大的去重指标会单独拆分到第2个列族。

![](../img/kylin-assign_model.jpg)

### 2.2 Doris的混合模型

Doris的聚合模型借鉴自Mesa，但本质上和Kylin的聚合模型一样，只不过Doris中将维度称作Key，指标称作Value。

![](../img/Doris-data-model.jpg)

Doris中比较独特的聚合函数是Replace函数，这个聚合函数能够**保证相同Keys的记录只保留最新的Value**，可以借助这个Replace函数来实现**点更新**。一般OLAP系统的数据都是只支持Append的，但是像电商中交易的退款，广告点击中的无效点击处理，都需要去更新之前写入的单条数据，在Kylin这种没有Relpace函数的系统中我们必须把包含对应更新记录的整个Segment数据全部重刷，但是有了Relpace函数，我们只需要再追加1条新的记录即可。 但是Doris中的Repalce函数有个缺点：**无法支持预聚合**，就是说只要你的SQL中包含了Repalce函数，即使有其他可以已经预聚合的Sum，Max指标，也必须现场计算。

为什么Doirs可以支持点更新呢？

Kylin中的Segment是不可变的，也就是说HFile一旦生成，就不再发生任何变化。但是Doirs中的Segment文件和HBase一样，是可以进行Compaction的。

Doris的聚合模型相比Kylin有个缺点：**就是一个Column只能有一个预聚合函数，无法设置多个预聚合函数**。 不过Doris可以现场计算出其他的聚合函数。 Apache Doris的开发者Review时提到，针对这个问题，Doris还有一种解法：由于**Doris支持多表导入的原子更新**，所以1个Column需要多个聚合函数时，可以在Doris中建多张表，同一份数据导入时，Doris可以同时原子更新多张Doris表，缺点是多张Doris表的查询路由需要应用层来完成。

Doris中和Kylin的Cuboid等价的概念是RollUp表，**Cuboid和RollUp表都可以认为是一种Materialized Views或者Index**。Doris的RollUp表和Kylin的Cuboid一样，**在查询时不需要显示指定**，系统内部会根据查询条件进行路由。 如下图所示：

![](../img/Doris-rollup.jpg)

Doris中RollUp表的路由规则如下：

- 选择包含所有查询列的RollUp表
- 按照过滤和排序的Column筛选最符合的RollUp表
- 按照Join的Column筛选最符合的RollUp表
- 行数最小的
- 列数最小的

### 2.3 Kylin Cuboid VS Doris RollUp

|              | Doris RollUp                                     | Kylin Cuboid                                                 |
| ------------ | ------------------------------------------------ | ------------------------------------------------------------ |
| 定义的成本   | 需要手动逐个定义                                 | 系统根据Web上维度，聚集组的设置自动定义出所有Cuboid          |
| 定义的灵活性 | 维度列和指标列可以自由选择                       | 只可以选择维度列，每个Cuboid都必须包含所有指标列             |
| 计算方式     | 从原数据直接生成每个RollUp表的数据               | 根据Cuboid Tree分层构建Cuboid，每个Cuborid的输入时Parent cuboid，不是原始数据 |
| 物理存储     | 每个RollUp是独立存储的                           | 多个Cuborid会存储到一个HFile中(按照大小)                     |
| 查询路由     | 会根据过滤列、排序列、Join列、行数、列数进行路由 | 仅会根据维度列进行路由                                       |

### 2.4 Doris的明细模型

由于Doris的聚合模型存在下面的缺陷，Doris引入了明细模型：

- 必须区分维度列和指标列
- 维度列很多时，Sort的成本很高
- Count成本很高，需要读取所有维度列（可以参考Kylin的解决方法进行优化）

Doris的明细模型不会有任何预聚合，不区分维度列和指标列，但是在建表时需要指定Sort Columns，**数据导入时会根据Sort Columns进行排序，查询时根据Sort Column过滤会比较高效**。

如下图所示，Sort Columns是Year和City。

![](../img/Doris-sort-column.jpg)

这里需要注意一点，**Doris中一张表只能有一种数据模型**，即要么是聚合模型，要么是明细模型，而且**Roll Up表的数据模型必须和Base表一致**，也就是说明细模型的Base表不能有聚合模型的Roll Up表