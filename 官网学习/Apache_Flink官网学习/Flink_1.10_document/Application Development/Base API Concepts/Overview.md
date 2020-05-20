# Basic API Concepts

Flink程序是常规程序，可对分布式集合进行转换操作(例如：过滤，映射，更新状态，联接，分组，定义窗口，聚合)。 集合是根据来源创建的(例如，通过读取文件，kafka主题或本地的内存中集合)。结果通过接收器返回，接收器可以例如将数据写入(分布式)文件或标准输出(例如，命令行终端)。 Flink程序可在各种上下文中运行，独立运行或嵌入其他程序中。 执行脚本可以在本地JVM或许多计算机的群集中进行。

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

Flink的数据模型不是基于键值对。 因此，您无需将数据集类型实际打包到键和值中。 key是“虚拟的”：定义为对实际数据指导分组操作的功能。

### 4.1 定义Tuples的Key

最简单的情况是在Tuple的一个或多个字段上对元组进行分组：

```java
DataStream<Tuple3<Integer,String,Long>> input = // [...]
KeyedStream<Tuple3<Integer,String,Long>,Tuple> keyed = input.keyBy(0) //元组根据第一个属性分组
```

```java
DataStream<Tuple3<Integer,String,Long>> input = // [...]
KeyedStream<Tuple3<Integer,String,Long>,Tuple> keyed = input.keyBy(0,1) //第一和第二个属性的复合键中的元组。
```

```java
DataStream<Tuple3<Tuple2<Integer, Float>,String,Long>> ds;
```

指定keyBy(0)将导致系统使用完整的Tuple2作为键（以Integer和Float为键）。 如果要定位到嵌套的Tuple2中，则必须使用字段表达式键，下面对此进行了说明。

### 4.2 Define keys using Field Expressions

可以使用基于字符串的字段表达式来引用嵌套字段，并定义用于分组，排序，联接或联合分组的键。

字段表达式使选择(嵌套)复合类型(例如Tuple和POJO类型)中的字段变得非常容易。

```java
// some ordinary POJO (Plain old Java Object)
public class WC {
  public String word;
  public int count;
}
DataStream<WC> words = // [...]
DataStream<WC> wordCounts = words.keyBy("word").window(/*window specification*/);
```

**Field Expression语法**

- 通过字段名选择POJO字段
- 通过字段名或从0开始频移的索引选择Tuple的字段
- Pojo或元组的嵌套字段，例如，“ user.zip”是指存储在POJO类型的“ user”字段中的POJO的“ zip”字段。 支持POJO和元组的任意嵌套和混合，例如“ f1.user.zip”或“ user.f3.1.zip”。
- 可以使用“ *”通配符表达式选择全部类型。 这对于非Tuple或POJO类型的类型也适用。

**Field Expression示例**

```java
public static class WC {
  public ComplexNestedClass complex; //nested POJO
  private int count;
  // getter / setter for private field (count)
  public int getCount() {
    return count;
  }
  public void setCount(int c) {
    this.count = c;
  }
}
public static class ComplexNestedClass {
  public Integer someNumber;
  public float someFloat;
  public Tuple3<Long, Long, String> word;
  public IntWritable hadoopCitizen;
}
```

可以选择：

- count
- complex
- complex.word.f2
- complex.hadoopCitizen

### 4.3 Define keys using Key Selector Functions

使用Key Selector方法定义键， 键选择器函数将单个元素作为输入，并返回该元素的键

```java
public class WC {public String word; public int count;}

KeyedStream<WC> keyed = words
  .keyBy(new KeySelector<WC, String>() {
     public String getKey(WC wc) { return wc.word; }
   });
```

## 5. Specifying Transformation Functions

许多转换方法需要用户自定义方法，本节列出了不同的定义方法。

### 5.1 Implementing an interface

```java
class MyMapFunction implements MapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
};
data.map(new MyMapFunction());
```

### 5.2 匿名内部类

```java
data.map(new MapFunction<String, Integer> () {
  public Integer map(String value) { return Integer.parseInt(value); }
});
```

### 5.3 Java8 Lambda表达式

Flink支持Java8 Lanbda 表达式。

```java
data.filter(s -> s.startsWith("http://"));

data.reduce((i1,i2) -> i1 + i2);
```

### 5.4 Rich functions

所有需要用户定义函数的转换都可以作为rich  function的参数。可以将

```java
class MyMapFunction implements MapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
};
```

替换为：

```java
class MyMapFunction extends RichMapFunction<String, Integer> {
  public Integer map(String value) { return Integer.parseInt(value); }
};
```

Rich function也可以被定义为内部类：

```java
data.map (new RichMapFunction<String, Integer>() {
  public Integer map(String value) { return Integer.parseInt(value); }
});
```

除了用户定义的功能(例如：map, reduce等)，rich function还提供了四个方法： `open`, `close`, `getRuntimeContext`, and `setRuntimeContext`。 这些对于参数化函数，创建和最终确定本地状态，访问广播变量，访问运行时信息(例如累加器和计数器) 以及迭代信息很有用。

## 6. 支持的数据类型

Flink对可以在DataSet或DataStream中的元素类型设置了一些限制。 原因是方便系统分析类型以确定有效的执行策略。

支持以下7种不同的数据类型：

1. **Java Tuples** and **Scala Case Classes**
2. **Java POJOs**
3. **Primitive Types**
4. **Regular Classes**
5. **Values**
6. **Hadoop Writables**
7. **Special Types**

### 6.1 Tuples and Case Classes

元组是复合类型，包含固定数量的各种类型的字段。 Java API提供了从Tuple1到Tuple25的类。 元组的每个字段可以是任意Flink类型，包括其他元组，从而导致嵌套元组。 可以使用字段名称tuple.f4或使用通用的getter方法tuple.getField（int position）直接访问元组的字段。 字段索引从0开始。请注意，这与Scala元组相反，但是与Java的常规索引更加一致。

```java
DataStream<Tuple2<String, Integer>> wordCounts = env.fromElements(
    new Tuple2<String, Integer>("hello", 1),
    new Tuple2<String, Integer>("world", 2));

wordCounts.map(new MapFunction<Tuple2<String, Integer>, Integer>() {
    @Override
    public Integer map(Tuple2<String, Integer> value) throws Exception {
        return value.f1;
    }
});

wordCounts.keyBy(0); // also valid .keyBy("f0")
```

### 6.2 POJOS

Java和scala类符合以下要求，则被Flink视为特殊的Pojo数据类型：

- 必须是public类
- 必须有公用的无参构造方法
- 所有字段是公共的，或者可以通过getter和setter函数访问
- 字段类型必须支持注册的序列化程序

POJO通常用PojoTypeInfo表示，并用PojoSerializer序列化（使用Kryo作为可配置的备用）。 当POJO实际上是Avro类型（Avro特定记录）或作为“ Avro反射类型”产生时，例外。 在这种情况下，POJO由AvroTypeInfo表示，并通过AvroSerializer进行序列化。 如果需要，还可以注册自己的自定义序列化程序；有关更多信息，请参见序列化。

```java
public class WordWithCount {

    public String word;
    public int count;

    public WordWithCount() {}

    public WordWithCount(String word, int count) {
        this.word = word;
        this.count = count;
    }
}

DataStream<WordWithCount> wordCounts = env.fromElements(
    new WordWithCount("hello", 1),
    new WordWithCount("world", 2));

wordCounts.keyBy("word"); // key by field expression "word"
```

## 7. 累加器与计数器

