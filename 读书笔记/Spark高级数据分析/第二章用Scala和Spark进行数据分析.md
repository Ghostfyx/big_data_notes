# 第二章 用Scala和Spark进行数据分析

数据清洗是数据科学项目的第一步，往往也是最重要的一步。许多灵巧的分析最后功败垂成，原因就是分析的数据存在严重的质量问题，或者数据中某些因素使分析产生偏见，或数据科学家得出根本不存在的规律。

数据科学家最为人称道的是在数据分析生命周期的每一个阶段都能发现有意思的，有价值的问题。在一个分析项目的早期阶段，投入的技能和思考越多，对最终的产品就越有信心。

## 2.1 数据科学家的Scala

对数据处理和分析，数据科学家往往都有自己钟爱的工具，比如R或者Python。为了能在R或Python中直接使用Spark，Spark上开发了专门的类库和工具包。Spark框架是用Scala语言编写的，采用与底层架构相同的编程语言有诸多好处：

- 性能开销小

	 为了能在基于JVM语言(比如Scala)上运行用于R或Python编写的算法，必须在不同环境中传递代码和数据，这会付出代价，而且在转换过程中信息时有丢失。

- 能用上最新的版本和最好的功能

- 有助于了解Spark原理

使用Spark和Scala做数据分析则是一种完全不同的体验，因为可以选择相同的言语完成所有的事情。借助Spark，用Scala代码读取集群上的数据，把Scala代码发送到集群上完成相同的转换，可以使用Spark高级API，使用Spark SQL注册UDF。在同一个环境中完成所有数据处理和分析，不用考虑数据本身在何处存放和在何处处理。

## 2.2 Spark编程模型

Spark编程模型始于数据集，而数据集往往存放在分布式持久化存储上，比如HDFS。编写Spark程序通常包括一些列相关步骤：

（1）在输入数据集上定义一组转换

（2）调用action，可以将转换后的数据集保存到持久化存储上，或者把结果返回到驱动器程序的本地存储

（3）运行本地计算，处理分布式计算结果，本地计算有助于确定下一步转换和action

要想理解Spark，就必须理解Spark框架提供的两种抽象：存储和执行。Spark优美的搭配这两类抽象，可以将数据处理管道中的任何中间步骤缓在内存里已备后用。

## 2.3 记录关联问题

问题大致情况如下：我们有大量来自一个或多个源系统的记录，其中多种不同的记录可能代码相同的基础实体：比如客户、病人、业务地址或事件。每个实体有若干属性，比如姓名、地址、生日。我们需要根据这些属性找到那些代表相同实体的记录，不幸的是，有些属性值有问题：格式不一致，或有笔误，或信息缺失，如果简单地对这些属性作相等性测试，则会漏掉许多重复记录。举例如下，表2-1列出的几家商店的记录。

​													**表2-1 记录关联问题的难点**

| 名称                         | 地址                     | 城市           | 州         | 电话           |
| ---------------------------- | ------------------------ | -------------- | ---------- | -------------- |
| Josh's Coffe Shop            | 1234 Sunset Boulevard    | West Hollywood | CA         | (213)-555-1212 |
| Josh Coffe                   | 1234 Sunset Blvd West    | Hollywood      | CA         | 555-1212       |
| Coffee Chain #1234           | 1400 Sunset Blvd #2      | Hollywood      | CA         | 206-555-1212   |
| Coffee Chain Regional Office | 1400 Sunset Blvd Suite 2 | Hollywood      | California | 206-555-1212   |

表中前两行其实指同一家咖啡店，但由于数据录入错误，这两项看起来是在不同城市，相反表后两行其实是同一咖啡连锁店的不同业务部门，尽管有相同的地址。

这个例子清楚地说明了记录关联为什么很困难：即使两组记录看起来很相似，但针对每组中的条目，我们确定重复的标准不一样，这种区别我们人类很容易理解，计算机却很难了解。

## 2.4 小试牛刀：Spark Shell和SparkContext

Spark-shell是Scala语言的一个REPL环境，同时针对Spark做了一些扩展。SparkContext负责协调集群上Spark作业的执行。

RDD是Spark提供的最基本的抽象，代表分布在集群中多台机器上的对象集合。Spark有两种方式可以创建RDD：

- 使用SparkContext基于外部数据源创建RDD，外部数据源包括HDFS上的文件、通过JDBC访问数据库表或Spark shell创建的本地对象集合
- 在一个或多个已有的RDD上执行转换操作来创建RDD，这些转换操作包括记录过滤、对具有相同键值的记录做汇总，把多个RDD关联在一起等

## 2.5 把数据集从集群上获取到客户端

使用RDD的first()、take()、collect()等方法可以将数据返回到客户端。

take()方法向客户端返回RDD的第一个元素，常用与对数据集做常规检查；collect()方法向客户端返回一个包含所有RDD内容的数组。take(n)方法向客户端返回指定数量的记录。

```scala
df.first()
df.collect()
df.take(n)
```

------

**动作**

创建RDD的操作并不会导致集群执行分布式计算。相反，RDD只是定义了作为计算过程中间步骤的逻辑数据集，只有调用RDD上的action(动作)时分布式计算才会执行。例如：count动作返回RDD中的记录个数：

```scala
rdd.count()
```

collect动作返回一个包含RDD中所有对象的Array(数组)：

```
rdd.collect()
```

动作不一定向本地进程返回结果，saveAsTextFile动作将RDD的内容保存到持久化存储(比如HDFS)上；

```scala
rdd.saveAsTextFile("hdfs://user/ds/mynumbers")
```

------

scala声明函数使用def关键字，必须为函数的参数指定类型，但是没必要指定函数的返回类型，原因在于Scala编译器能根据方法计算逻辑推断函数返回类型：

```scala
  def isHeader(line:String) = line.contains("id_1")
```

Scala也支持显式的指定返回类型，特别是在函数体很长，代码复杂并且包含多个return语句的情况，这时候，Scala编译器不一定能推断出函数的返回类型，为了函数代码可读性更好，也可以指明函数的返回类型。

```scala
def isHeader2(line:String):Boolean = {
    line.contains("id_1")
  }
```

Scala的匿名函数有点类似于Python的Lambda函数，为了减少函数输入，比如在匿名函数的定义中，为了定义匿名函数并给参数指定名称，只输入了字符`x =>`，Scala也允许使用下划线(_)表示匿名函数的参数：

```scala
head.filter(x => !isHeader(x))
head.filter(!isHeader(_))
```

## 2.6 把代码从客户端发送到集群

之前执行的代码都左右在head数组中的数据上，这些数据都在客户端机器上。现在，我们打算在Spark里把刚写好的代码应用关联到记录数据集RDD rawblocks，该数据集在集群上的记录有几百万条。用于在过滤集群上的语法和本地机器上的语法一样。这正是Spark的强大之处。

它意味着我们可以先从集群中采样得到小的数据集，在小数据集上开发和调试数据处理代码，等一切就绪后再把代码发送到集群上处理完整的数据集就可以了。

## 2.7 从RDD到DataFrame

我们遇到的大部分数据集都有着合理的结构，要么因为它本来就是如此，要么因为已经有人已经对数据做好了清洗和结构化，我们没必要对花费精力自己写一套代码解析，只要简单地调用现成的类库，并利用数据的结构，即可解析成所需的结构，Spark 1.3引入了一个这样的数据结构——DataFrame。

DataFrame是一个建立在RDD之上的Spark抽象，专门为结构规整的数据集而设计，DataFrame的一条记录就是一行，每一行由若干个列组成，每一列的数据类型都有严格的定义，可以把DataFrame类型实例理解为Spark版本的关系数据库表。DataFrame这个名字可能会联想到R语言的data.frame对象，或者Python的pandas.DataFrame对象，但是Spark的DataFrame与它们有很大的不同，因为Spark的DataFrame对象代表一个分布式数据集，而不是所有数据都存储在同一台机器上的本地数据。

要为记录关联数据集创建一个DataFrame，需要用到SparkSession对象，SparkSession是SparkContext对象的一个封装，可以通过SparkSession直接访问到SparkContext：

```scala
sparkSession.sparkContext()

val preview = spark.read.csv("hdfs:///user/ds/linkage")
preview.show()
preview.printSchema()
```

每个StuctField示例包含了列名，每条记录中数据的最具体的类型，以及一个表示此列是否允许控制的布尔字段(默认为true)。为了完成模式推断，Spark需要便利数据集两次：第一次找出每列的数据类型；第二次才真正进行解析，如果预先知道某个文件的模式，可以创建一个org.apache.spark.sql.types.StructType实例，并使用模式函数将它传给Reader API，在数据量很大的情况下，性能会得到巨大的吉他声，因为Spark不需要为确定每列数据类型而额外遍历一次数据。

------

**DataFrame与数据格式**

通过DataFrameReader和DataFrameWriter API，Spark 2.0内置支持多种格式读写DataFrame：

**json**

支持CSV格式的模式推断

**parquet**和 **orc**

两种二进制列存储格式，这两种格式可以相互替代

**jdbc**

通过JDBC数据连接标准连接到关系型数据库

**libsvm**

一种常用语表示特征稀疏并且带有标号信息的数据集的文本格式

**text**

文件的每行作为字符串整体映射到DataFrame一行

要访问DataFrameReader API中的方法，可以调用SparkSession实例的read方法。要从人间中加载数据，可以调用format和load方法，也可以使用更快捷的方法：

```scala
val jsonDf_1 = spark.read.format("json").load("file.json")
val josnDf_2 = spark.read.json("file.json")
```

如果要把数据导出，可以调用任何DataFrame实例的write方法访问DataFrameWriter API。DataFrameWriter API支持与DataFrameReader API相同的内置格式：

```scala
val jsonWriter_1 = jsonDf_1.write.json("file.json")
val jsonWriter_2 = jsonDf_1.write.format("json").save("file.json")
```

## 2.8 使用DataFrame API来分析数据

Spark的RDD API为分析数据提供了少量易用的方法，例如`count()`方法可以计算一个RDD包含的记录数，`countByValue()`方法可以获取不同值的分布直方图，`RDD[Double]`的`stats()`方法可以获取一些概要统计信息，例如最大值、最小值、平均值和标准差，但是DataFrame API的工具比RDD API更强大。

研究下DataFrame对象实例parsed的模式，看一下前几行数据，可以看到以下特征：

- 前两个字段是整型ID，代表在记录中匹配到的患者
- 后面9个值是数值类型(双精度浮点数或整型，可能有缺失值)，代表患者记录数据中不同字段的匹配得分值
- 最后一个字段是布尔值，表示这条记录重点的一对患者是否匹配。

目前为止，每次处理数据集中的数据时，Spark得重新打开文件，重新解析每一行，然后才能执行所需的操作。例如显示几行或计算记录总数，当需要执行另外一个操作时，Spark会反复执行读取及解析操作。

这种方式浪费了计算资源，数据一旦被解析完，就可以把解析后的数据保存在集群中，这样就不必每次都重新解析数据了。Spark支持这种用例，它允许调用cache方法，告诉RDD或DataFrame在创建时将它缓存在内存中。

```scala
parsed.cache()
```

------

**缓存**

虽然默认情况下DataFrame和RDD的内容是临时的，但是Spark提供了一种持久化底层数据的机制：

```scala
cached.cache()
cached.count()
cached.take()
```

在上述代码中，调用cache方法指示在下次计算DataFrame时，要把DataFrame的内容缓存起来，当调用take时，访问的是缓存，而不是从cached的依赖关系中重新计算出来的。

Spark为持久化数据定义了几种不同的机制，用不同的StorageLevel值表示。cache()是persist(StorageLevel.Memory)的简写，它将所有Row对象存储为未序列化的Java对象，当Spark的预计内存不同存放一个分区时，则不会在内存中存储这个分区，这样在下次计算时就必须重新计算，在对象需要频繁访问或低延迟访问时，适合使用StorageLevel.MEMORY，因为它可以避免序列化的开销，相比其他选项，StorageLevel.MEMORY的问题是要占用更大的内存空间，另外，大量小对象会对Java的垃圾回收施加压力，会导致程序停顿和常见的处理速度缓慢问题。

Spark也提供了MEMORY_SER的存储级别，用于在内存中分配大字节缓冲区，以存储记录的序列化内容，如果使用妥当(后续会详细介绍)，序列化数据占用的空间往往为未序列化数据的17%～33%。

Spark也可以用磁盘来缓存数据，存储级别MEMORY_AND_DISK和MEMORY_AND_DISK_SER分别类似于MEMORY和MEMORY_SER。对于MEMORY和MEMORY_SER，如果一个分区在内存中放不下，整个分区都不会放入内存，对于MEMORY_AND_DISK和MEMORY_AND_DISK_SER，如果分区在内存放不下，Spark会溢写到磁盘上。

虽然DataFrame和RDD都可以被缓存，但是有了DataFrame的模式信息，Spark就可以利用数据的详细信息，帮助DataFrame在持久化数据时达到比使用RDD的Java 对象高的多的效率。

决定何时缓存数据是一门艺术，这个决定通常涉及空间和速度之间的权衡，而且还需要是不是收到垃圾收集器的影响，因此如何抉择时很复杂的事情。一般来说，当数据可能被多个操作依赖时，并且相对于集群可用的内存和磁盘空间而言，如果数据集较小，而且重新生成代价很高，那么数据就应该被缓存起来。

------

DataFrame封装的RDD由Row的实例组成，包括通过索引位置(从0开始计数)获取每个记录中值的访问方法，以允许通过名称查找给定类型的字段`getAs[T]`方法。

注意：有两种方式引用DataFrame的列名：作为字面量引用，例如`groupBy("is_match")`；作为Column对象引用，