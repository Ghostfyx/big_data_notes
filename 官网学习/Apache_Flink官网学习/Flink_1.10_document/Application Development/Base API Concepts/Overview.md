# Basic API Concepts

Flink程序是常规程序，可对分布式集合进行转换操作(例如：过滤，映射，更新状态，联接，分组，定义窗口，聚合)。 集合是根据来源创建的(例如，通过读取文件，kafka主题或本地的内存中集合)。结果通过接收器返回，接收器可以例如将数据写入（分布式）文件或标准输出（例如，命令行终端）。 Flink程序可在各种上下文中运行，独立运行或嵌入其他程序中。 执行脚本可以在本地JVM或许多计算机的群集中进行。

根据数据源的类型(即有界或无界源)，您将编写批处理程序或流式程序，其中将DataSet API用于批处理，将DataStream API用于流式处理。 本指南将介绍两个API共有的基本概念，

## 1. DataSet and DataStream

Flink具有特殊的类DataSet和DataStream来表示程序中的数据。 可以将它们视为包含重复项的不可变数据集合。 在使用DataSet的情况下，数据是有限的，而对于DataStream而言，元素的数量是不受限制的。

这些集合在某些关键方面与常规Java集合不同。 首先，它们是不可变的，这意味着一旦创建它们就不能添加或删除元素。 也不能简单地检查其中的元素(类似于Spark的RDD)。

通过在Flink程序中添加source来创建集合的，然后使用诸如map，filter等的API方法对它们进行转换，从而从中获得新的集合。

## 2. 剖析Flink项目

Flink程序看起来像转换数据集合的常规程序。 每个程序都包含相同的基本部分：

- 获得运行环境(environment)
- 加载/创建初始数据
- 指定数据转换
- 指定计算结果存储介质
- 触发程序执行

注意，在包`org.apache.flink.api.java`中可以找到Java DataSet API的所有核心类，而在`org.apache.flink.streaming.api`中可以找到Java DataStream API的类。下面对上述每个部分进行介绍：

### 2.1 ExecutionEnvironment

StreamExecutionEnvironment是所有Flink程序的基础。 可以在StreamExecutionEnvironment上使用以下静态方法来获得一个运行环境：

```java
getExecutionEnvironment()

createLocalEnvironment()

createRemoteEnvironment(String host, int port, String... jarFiles)
```

通常情况下，只需要使用`getExecutionEnvironment()`，因为这将根据上下文执正确的初始化操作：如果在IDE中执行程序或作为常规Java程序执行，它将创建一个本地环境，该环境将本地计算机上执行程序。如果是从程序中打包的JAR文件，并提交Jar包到Flink集群，集群管理器(例如：YARN，Messo等)将执行指定的main方法，getExecutionEnvironment()则将返回用于在集群上执行程序的执行环境。

### 2.2 DataSource

Flink支持从多种类型数据源读取有界/无界数据。例如所示：按行读取文本中数据。

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

DataStream<String> text = env.readTextFile("file:///path/to/file");
```

### 2.4 Data Transform

```java
DataStream<String> input = ...;

DataStream<Integer> parsed = input.map(new MapFunction<String, Integer>() {
    @Override
    public Integer map(String value) {
        return Integer.parseInt(value);
    }
});
```

### 2.5 Data Sink

指定数据存储介质，Flink支持HDFS、kafka、ES等存储介质。

```java
writeAsText(String path)
```

### 2.6 ExecutionEnvironment execute操作

构建完毕完整处理程序，需要调用 `execute()`上`StreamExecutionEnvironment`**触发执行程序**。根据执行的类型，`ExecutionEnvironment`执行将在本地计算机上触发或将您的程序提交到群集上执行。

`execute()`方法将等待作业完成，然后返回`JobExecutionResult`，其中包含执行时间和累加器结果。

```java
public JobExecutionResult execute() throws Exception {
	return execute(DEFAULT_JOB_NAME);
}
```

如果不想等待作业完成，可以通过调用触发异步作业执行的方法 `executeAysnc()`。它将返回一个`JobClient`可以与刚提交的作业进行通信。

```java
final JobClient jobClient = env.executeAsync();
```

## 3. Lazy Evaluation

所有Flink程序都是延迟执行的：执行程序的main方法时，不会直接进行数据加载和转换。而是会创建每个操作并将其添加到程序的计划中。当在执行环境中由`execute()`调用显式触发执行时，实际上将执行这些操作。程序是在本地执行还是在群集上执行取决于执行环境的类型。

Lazy Evaluation加载机制可以构建复-杂的程序，Flink将其作为一个整体计划的单元来执行。

## 4. 指定键值

某些转换(join，coGroup，keyBy，groupBy)要求在元素集合上定义键。 其他转换(Reduce，GroupReduce，Aggregate，Windows)允许在应用数据之前对数据进行分组。

DataSet上指定键分组：

```java
DataSet<...> input = // [...]
DataSet<...> reduced = input
  .groupBy(/*define key here*/)
  .reduceGroup(/*do something*/);
```

DataStream指定键分组：

```java
DataStream<...> input = // [...]
DataStream<...> windowed = input
  .keyBy(/*define key here*/)
  .window(/*window specification*/);
```

