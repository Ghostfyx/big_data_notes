# 第三章 RDD编程

本章介绍Spark对数据的核心抽象RDD——弹性分布式数据集（Resilient Distributed Dataset），简称RDD。RDD本质上是分布式的元素集合，在Spark中，对数据的操作不外乎创建RDD，转换RDD和调用RDD值进行求值，一切操作的背后，是Spark将RDD中的数据发送在集群上，将操作并行化执行。

## 3.1 RDD基础

Spark中的RDD是一个不可变的分布式对象集合，每个RDD都被分为多个分区，这些分区运行在集群中的不同节点上，RDD可以包含Python，Java，Scala中的任意类型对象。

用户可以通过两种方式创建RDD：

- 读取外部数据集
- 驱动程序中的对象集合

RDD支持两种类型的操作：

- 转换操作：由一个RDD生成一个新的RDD；
- 行动操作：对RDD计算出一个结果，并把结果返回到驱动器程序中，或存储在外部存储系统，如HDFS。

转换操作和行动操作的区别在于Spark计算RDD的方式不同，用户可以在任何时候定义RDD，但是Spark只会惰性计算RDD，只有在第一次行动操作中用到才会真正的计算。这样的计算适用于大数据领域，如果每次创建和转换RDD的时候立即执行，就会将每次运行的数据存储起来，消耗很多存储空间（这也是Spark速度远快于MapReduce的原因）。

默认情况下，Spark的RDD会在每次进行行动操作时重新计算，如果在多个行动中重用同一个RDD，可以使用RDD.persist( ) 方法持久化RDD。Spark可以将RDD持久化在多个存储介质中，在第一次对持久化的RDD进行计算后，Spark会将RDD数据在内存中（以分区保存在集群上），后续计算会快速读取和计算（减少磁盘寻址，数据加载的时间）。**注意：**如果不需要多次使用同一个RDD，没有必要持久化，浪费存储空间。

在实际操作中，经常会使用persit( )方法将部分数据保存在内存中。总体来说，Spark程序或脚本都会按照下面方式工作：

- 外部数据创建输入RDD
- 使用诸如filter( )等转换操作将输入RDD转化
- 对需要重用的中间结果调用persist( )方法持久化操作
- 使用行动操作触发一次并行化计算，Spark会对计算进行优化后再执行

## 3.2 创建RDD

Spark提供两种创建RDD的方法：

- 外部数据集读取；
- 驱动程序中对一个集合进行并行化

### 3.2.1 并行化创建

把程序中的已有集合传给SparkContext的parellelize( )方法。一般在实际开发和测试中不会使用。

### 3.2.2 外部数据源

详细内容在第五章详细描述

## 3.3 RDD操作

RDD支持转换操作和行动操作，两者区别是行动操作会进行实际计算，将计算结果返回驱动器程序或写入外部存储结果，而且接口API返回值为非RDD的其他数据类型；转换操作是将RDD转换为RDD，接口API的返回值为RDD。

### 3.3.1 转换操作

转换操作几个关键知识点：

- 许多转换操作时针对RDD分区进行操作；
- 转换操作可以操作任意数量的输入RDD；
- 通过转换操作，从已有的RDD中派生出新的RDD，Spark会用谱系图（lineage graph）记录不同RDD之间的依赖关系（宽依赖与窄依赖）。Spark需要用这些依赖信息按需计算每个RDD，也可以在持久化的RDD丢失部分数据时重新计算，恢复丢失数据（例如：部分节点异常，持久化数据丢失，在其他节点并行化计算，将缺失数据重新存储在其他节点中）。

### 3.3.2 行动操作

行动操作几个关键知识点：

- 对RDD进行计算，返回结果或选择将其存储在外部存储介质中，例如：HDFS，Amazon S3；
- collect( )方法在Driver中执行，因此当单台机器内容存放的下数据集时才能使用，不建议在生产中使用，可用与小规模数据集的调试；
- 适当缓存RDD计算的中间结果，因为每次执行行动操作，整个RDD关系链都会从头计算，这样可以避免浪费资源；

### 3.3.3 惰性求值

几个关键知识点：

- RDD在调用行动操作前不会计算，例如：文件读取时，RDD不会立即将文件加载到内存；
- Spark会在内部记录下所有要求执行操作的相关信息，RDD不仅是数据存储结构，而且是记录着计算数据的指令列表；

惰性求值优点：

- 把一些操作合并减少计算步骤；
- 减少文件读写消耗的时间资源和硬件资源。例如在Hadoop MapReduce系统中，需要将中间执行结果临时存储在磁盘中，下一阶段再读取，MapReduce若操作阶段过多，会导致高延时，需要花费大量时间优化；而在Spark中，复杂映射不一定比多个简单连续操作性能更好，用户可以更好关注与数据处理。

## 3.4 向Spark传递函数

Spark 的大部分转化操作和一部分行动操作，都需要依赖用户传递的函数来计算。在我们 支持的三种主要语言中，向 Spark 传递函数的方式略有区别。

### 3.4.1 Python

在 Python 中，我们有三种方式来把函数传递给 Spark。

- 传递比较短的函数时，可以使用 lambda 表达式来传递

	```python
	# 例 3-18：在 Python 中传递函数 
	word = rdd.filter(lambda s: "error" in s) 
	```

- 传递局部函数或顶层函数

	```python
	def containsError(s):     
		return "error" in s 
	word = rdd.filter(containsError)
	```

注意：Python 会在你不经意间把函数所在的对象也序列化传出 去。当你传递的对象是某个对象的成员，或者包含了对某个对象中一个字段的引用时（例 如 self.field），Spark 就会把整个对象发到工作节点上，这可能比你想传递的东西大得多 。代码示例如下：

```python
# 传递一个带字段引用的函数（别这么做！） 
class SearchFunctions(object):  
    def __init__(self, query):       
        self.query = query   
    def isMatch(self, s):       
        return self.query in s   
    def getMatchesFunctionReference(self, rdd):       
        # 问题：在"self.isMatch"中引用了整个self       
        return rdd.filter(self.isMatch)   
```

替代的方案是，只把你所需要的字段从对象中拿出来放到一个局部变量中，然后传递这个 局部变量。

```python
class WordFunctions(object):   
    def getMatchesNoReference(self, rdd):       
        # 安全：只把需要的字段提取到局部变量中       
        query = self.query      
        return rdd.filter(lambda x: query in x)
```

### 3.4.2 Scala

待学习完毕Scala补充；

### 3.4.3 java

在 Java 中，函数需要作为实现了 Spark 的 org.apache.spark.api.java.function 包中的任 一函数接口的对象来传递。根据不同的返回类型，定义了一些不同的接口。

​																表3-1：标准Java函数接口

| 函数名 实现的方法 用途 | 实现的方法          | 用途                                                         |
| ---------------------- | ------------------- | ------------------------------------------------------------ |
| Function<T, R>         | R call(T)           | 接收一个输入值并返回一个输出值，用于类似 map() 和 filter() 等操作中 |
| Function<T1,T2, R>     | R call(T1, T2)      | 接收两个输入值并返回一个输出值，用于类似 aggregate() 和 fold() 等操作中 |
| FlatMapFunction<T, R>  | Iterable<R> call(T) | 接收一个输入值并返回任意个输出，用于类似 flatMap() 这样的操作中 |

可以把我们的函数类内联定义为使用匿名内部类.。

```java
RDD<String> errors = lines.filter(new Function<String, Boolean>() {   
    public Boolean call(String x) { return x.contains("error"); 
     } 
});

```

也可以创建一个具名类。

```java
class ContainsError implements Function<String, Boolean>() {   
    public Boolean call(String x) { return x.contains("error"); 
                                  } } 
 RDD<String> errors = lines.filter(new ContainsError());
```

在Java8中使用Lambda表达式简单实现函数式接口（推荐！）。

## 3.5 常见转换操作和行动操作

本节以及后面几节常见的转化操作和行动操作。。包含特定数据类型的 RDD 还支持一些附加操作，例如，包含数字类型的 RDD 支持统计型函数操作，而键值对形式的 RDD 则支持诸如根据键聚合数据的键值对操作。

### 3.5.1 基本RDD

首先讲下哪些转化操作和行动操作受任意数据类型RDD支持。

#### 1. 针对各个元素的转换操作

- map( )操作：接收一个函数，把这个函数用于RDD中的每一个元素，将函数的返回结果作为结果RDD中对应元素的值。
- filter( )操作：接收一个函数，将原RDD中满足该函数的元素放入新的RDD中返回。
- flatMap( )操作：接收一个函数，返回值序列的迭代器

![](./img/3-3.jpg)

​														**图 3-3：RDD 的 ﬂatMap() 和 map() 的区别**

​											表3-2 对一个数据为$\{1,2,3,3\}$的RDD就行基本的RDD转换

| 函数名                                        | 目的                                                         |
| --------------------------------------------- | ------------------------------------------------------------ |
| map(func)                                     | 将元素应用于RDD中的每个元素，将返回值构成新的RDD             |
| flatMap(func)                                 | 将元素应用于RDD的每个元素，将返回迭代器的元素构成新的RDD。   |
| filter(*func*)                                | 条件过滤函数，返回通过过滤元素组成的新RDD                    |
| distinct( )                                   | 去重                                                         |
| sample(*withReplacement*, *fraction*, *seed*) | 数据采样，有三个可选参数：设置是否放回（withReplacement）、采样的百分比（*fraction*）、随机数生成器的种子（seed） |

#### 2. 伪集合操作

RDD本身不是严格意义上的集合，但是支持数学上的集合操作。例如交集，并集。

![](./img/3-4.jpg)

​															**图 3-4：一些简单的集合操作**

 									表3-3:对数据分别为{1, 2, 3}和{3, 4, 5}的RDD进行针对两个RDD的转化操作 

| 函数名                       | 目的                                       |
| ---------------------------- | ------------------------------------------ |
| union(*otherDataset*)        | 生成一个包含两个 RDD 中所有元素的 RDD      |
| intersection(*otherDataset*) | 求两个 RDD 共同的元素的 RDD，两个RDD的并集 |
| subtract(*otherDataset*)     | 移除一个 RDD 中的内容(例如移除训练数据)    |
| cartesian(*otherDataset*)    | 与另一个 RDD 的笛卡儿积                    |

#### 3. 行动操作

RDD 的一些行动操作会以普通集合或者值的形式将 RDD 的部分或全部数据返回驱动器程序中。把数据返回驱动器程序中最简单、最常见的操作是collect()，它会将整个 RDD 的内容返 回。collect() 通常在单元测试中使用，因为此时 RDD 的整个内容不会很大，可以放在内 存中。使用 collect() 使得 RDD 的值与预期结果之间的对比变得很容易。由于需要将数据 复制到驱动器进程中，collect() 要求所有数据都必须能一同放入单台机器的内存中。 

   												**表3-4:对RDD进行基本的RDD行动操作** 

| 函数名                                   | 目的                                                         |
| ---------------------------------------- | ------------------------------------------------------------ |
| collect( )                               | 返回 RDD 中的所有元素                                        |
| count()                                  | RDD 中的元素个数                                             |
| countByValue()                           | 各元素在 RDD 中出现的次数                                    |
| take(num)                                | 从 RDD 中返回 num 个元素                                     |
| top(num)                                 | 从 RDD 中返回最前面的 num 个元素                             |
| takeOrdered(num) (ordering)              | 从 RDD 中按照提供的顺序返 回最前面的 num 个元素              |
| takeSample(withReplacement, num, [seed]) | 从 RDD 中返回任意一些元素                                    |
| reduce(func)                             | 并行整合 RDD 中所有数据 (例如 sum)                           |
| fold(zero)(func)                         | 和 reduce() 一样，但是需要提供初始值，先将初值应用于RDD元素中，在执行func操作 |
| aggregate(zeroValue) (seqOp, combOp)     | 和 reduce() 相似，但是通常 返回不同类型的函数                |
| foreach(func)                            | 对 RDD 中的每个元素使用给 定的函数                           |

### 3.5.2 在不同RDD类型间的转换

有些函数只能用于特定类型的 RDD，比如 mean() 和 variance() 只能用在数值 RDD 上， 而 join() 只能用在键值对 RDD 上。 在 Scala 和 Java 中，这些函数都没有定义在标准的 RDD 类中，所以要访问这些附加功能，必须要确保获得了正确的专用 RDD 类。 

#### 1. Scala

在 Scala 中，将 RDD 转为有特定函数的 RDD（比如在 RDD[Double] 上进行数值操作）是 由隐式转换来自动处理的。需要加上 import org.apache.spark. SparkContext._ 来使用这些隐式转换。

#### 2. Java

在 Java 中，各种 RDD 的特殊类型间的转换更为明确。Java 中有两个专门的类 JavaDoubleRDD 和 JavaPairRDD，来处理特殊类型的 RDD，这两个类还针对这些类型提供了额外的函数。 这让你可以更加了解所发生的一切，但是也显得有些累赘。 

在 Java 中，各种 RDD 的特殊类型间的转换更为明确。Java 中有两个专门的类 JavaDoubleRDD 和 JavaPairRDD，来处理特殊类型的 RDD，这两个类还针对这些类型提供了额外的函数。 

 										表3-5:Java中针对专门类型的函数接口 

| 函数名                       | 等价函数                            | 用途                                      |
| ---------------------------- | ----------------------------------- | ----------------------------------------- |
| DoubleFlatMapFunction<T>     | Function<T, Iterable<Double>>       | 用于 flatMapToDouble，以生成 DoubleRDD    |
| DoubleFunction<T>            | Function<T, Double>                 | 用于 mapToDouble，以生成 DoubleRDD        |
| PairFlatMapFunction<T, K, V> | Function<T, Iterable<Tuple2<K，V>>> | 用于 flatMapToPair，以生 成 PairRDD<K, V> |
| PairFunction<T, K, V>        | Function<T, Tuple2<K，V>>           | 用 于 mapToPair， 以 生 成 PairRDD<K, V>  |

## 3.6 持久化（缓存）

Spark RDD 是惰性求值的，而有时我们希望能多次使用同一个 RDD。如果简单 地对 RDD 调用行动操作，Spark 每次都会重算 RDD 以及它的所有依赖。这在**迭代算法**中消耗格外大，因为迭代算法常常会多次使用同一组数据。 

为了避免多次计算同一个 RDD，可以让 Spark 对数据进行持久化。当我们让 Spark 持久化 存储一个 RDD 时，计算出 RDD 的节点会分别保存它们所求出的分区数据。如果一个有持 久化数据的节点发生故障，Spark 会在需要用到缓存的数据时重算丢失的数据分区。如果 希望节点故障的情况不会拖累我们的执行速度，也可以把数据备份到多个节点上。 

出于不同的目的，可以为RDD选择持久化级别，在Scala和Java中，默认情况persist( )会将数据以序列化的形式缓存在JVM的堆空间中。在 Python 中，我们会始终序列化要持久化存储的数据，所以持久化级别默认值就是以序列化后的对象存储在 JVM 堆空间中。 当我们把数据写在堆外内存或者外部磁盘的时候，也总是使用序列化后的数据。

表3-6:org.apache.spark.storage.StorageLevel和pyspark.StorageLevel中的持久化级 别;如有必要，可以通过在存储级别的末尾加上“_2”来把持久化数据存为两份 

| 级别                | 使用空间 | CPU时间 | 是否在内存中 | 是否在磁盘上 | 备注                                                         |
| ------------------- | -------- | ------- | ------------ | ------------ | ------------------------------------------------------------ |
| MEMORY_ONLY         | 高       | 低      | 是           | 否           |                                                              |
| MEMORY_ONLY_SER     | 低       | 高      | 是           | 否           |                                                              |
| MEMORY_AND_DISK     | 高       | 中等    | 部分         | 部分         | 若数据内存存储空间不足，则溢写在磁盘中                       |
| MEMORY_AND_DISK_SER | 低       | 高      | 部分         | 部分         | 若数据内存存储空间不足，则溢写在磁盘中，内存存放序列化后的数据 |
| DISK_ONLY           | 低       | 高      | 否           | 是           |                                                              |

```scala
import org.apache.spark.storage.StorageLevel
val result = input.map(x => x * x)
result.persist(StorageLevel.DISK_ONLY)
println(result.count())
println(result.collect().mkString(","))
```

如果要缓存的数据太多，内存中放不下，Spark会自动利用最近最少使用（LRU）的缓存策略从内存中移除部分分区数据，下一次要用到 已经被移除的分区时，这些分区就需要重新计算。但是对于使用内存与磁盘的缓存级别的 分区来说，被移除的分区都会写入磁盘。 

RDD 还有一个方法叫作 unpersist()，调用该方法可以手动把持久化的 RDD 从缓 存中移除。 