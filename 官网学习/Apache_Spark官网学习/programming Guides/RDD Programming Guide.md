# RDD编程指南

## 1. Overview

在较高级别上，每个Spark应用程序都包含一个驱动程序，该程序运行用户的主要功能并在集群上执行各种并行操作。 Spark提供的主要抽象是弹性分布式数据集(RDD)，它是跨集群节点划分元素的集合，可以并行操作。通过Hadoop文件系统(或者任何其他Hadoop支持的文件系统)或驱动程序中现有的Scala集合开始并进行转换来创建RDD。 用户还可以要求Spark将RDD保留在内存中，以使其能够在并行操作中有效地重用。 最后，RDD自动从节点故障中恢复。

Spark中的第二个抽象是可以在并行操作中使用的共享变量(shared variables)。默认情况下，当Spark作为一组任务在不同节点上并行运行一个函数时，它会将函数中使用的每个变量的副本传送给每个任务。 有时，需要在任务之间或任务与驱动程序之间共享变量。 Spark支持两种类型的共享变量：广播变量(可用于在所有节点上的内存中缓存值)和累加器(累加器)是仅“添加”到变量的变量，例如计数器和总和。

## 2. Linking with Spark

maven依赖如下：

```
groupId = org.apache.spark
artifactId = spark-core_2.12
version = 2.4.5
```

将Spark类导入到程序中

```java
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.api.java.JavaRDD;
import org.apache.spark.SparkConf;
```

## 3. Initializing Spark

Spark程序必须做的第一件事是创建一个[JavaSparkContext](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/api/java/JavaSparkContext.html)对象，该对象告诉Spark如何访问集群。要创建一个，`SparkContext`首先需要构建一个[SparkConf](https://spark.apache.org/docs/latest/api/java/index.html?org/apache/spark/SparkConf.html)对象，其中包含有关应用程序的信息。

```java
SparkConf conf = new SparkConf().setAppName(appName).setMaster(master);
JavaSparkContext sc = new JavaSparkContext(conf);
```

`appName`参数是应用程序显示在集群UI上的名称，master是Spark Master、Mesos、YARN集群的URL或“local”，以本地模式运行。

## 4. Using the Shell

在Spark shell中，已经在名为sc的变量中创建了一个特殊的可识别解释器的SparkContext。 制作自己的SparkContext将不起作用。 可以使用--master参数设置上下文连接的主机，也可以通过将逗号分隔的列表传递给--jars参数来将JAR添加到类路径。 还可以通过在--packages参数中提供逗号分隔的Maven坐标列表，从而将依赖项(例如Spark Packages)添加到Shell会话中。 可以存在依赖项的任何其他存储库（例如Sonatype）都可以传递给--repositories参数。 例如，要在正好四个内核上运行bin / spark-shell，请使用：

```
$ ./bin/spark-shell --master local[4]
```

添加`code.jar`到其类路径中，使用：

```
$ ./bin/spark-shell --master local[4] --jars code.jar
```

## 5. Resilient Distributed Datasets (RDDs)

Spark围绕弹性分布式数据集(RDD)的概念展开，RDD是可并行操作的元素的容错集合。创建RDD的方法有两种：

- 在Driver程序中并行化集合
- 引用外部存储系统(例如共享文件系统、HDFS、HBase或提供Hadoop InputFormat的任何数据源)中的数据集

### 5.1 Parallelized Collections

通过在驱动程序中现有的上调用`JavaSparkContext`的`parallelize`方法来创建并行集合`Collection`。复制集合的元素以形成可以并行操作的分布式数据集。例如，以下是创建包含数字1到5的并行化集合的方法：

```java
List<Integer> data = Arrays.asList(1, 2, 3, 4, 5);
JavaRDD<Integer> distData = sc.parallelize(data);
```

创建后，分布式数据集(`distData`)可以并行操作。例如，可能会调用`distData.reduce((a, b) -> a + b)`以添加列表中的元素。稍后将描述对分布式数据集的操作。

