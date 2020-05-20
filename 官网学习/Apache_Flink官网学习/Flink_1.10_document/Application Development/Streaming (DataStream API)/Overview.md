# Flink DataStream API Programming Guide

Flink中的DataStream程序是常规程序，可对数据流进行转换(例如，过滤，更新状态，定义窗口，聚合)。 最初从各种来源(例如，消息队列，套接字流，文件)创建数据流。 结果通过接收器返回，接收器可以例如将数据写入文件或标准输出(例如命令行终端)。 Flink程序可以在各种上下文中运行，独立运行或嵌入其他程序中。 执行可以在本地JVM或许多计算机的群集中进行。

## 1. 示例代码

以下程序是流式窗口单词计数应用程序的一个完整的工作示例，该应用程序在5秒的窗口中对来自Web Socket的单词进行计数。 可以复制并粘贴代码以在本地运行。

```java
import org.apache.flink.api.common.functions.FlatMapFunction;
import org.apache.flink.api.java.tuple.Tuple2;
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.util.Collector;

public class WindowWordCount {

    public static void main(String[] args) throws Exception {

        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        DataStream<Tuple2<String, Integer>> dataStream = env
                .socketTextStream("localhost", 9999)
                .flatMap(new Splitter())
                .keyBy(0)
                .timeWindow(Time.seconds(5))
                .sum(1);

        dataStream.print();

        env.execute("Window WordCount");
    }

    public static class Splitter implements FlatMapFunction<String, Tuple2<String, Integer>> {
        @Override
        public void flatMap(String sentence, Collector<Tuple2<String, Integer>> out) throws Exception {
            for (String word: sentence.split(" ")) {
                out.collect(new Tuple2<String, Integer>(word, 1));
            }
        }
    }

}
```

要运行示例程序，请首先从终端使用netcat启动输入流：

```
nc -lk 9999
```

## 2. Data Sources

Sources是应用程序读取数据输入的位置，可以使用以下方法将数据源添加到应用程序：

```java
StreamExecutionEnvironment.addSource(sourceFunction)
```

Flink附带了许多预先实现的源函数，但是用户可以通过实现`SourceFunction`接口定义非并行源，或通过实现`ParallelSourceFunction`接口定义并行源，或通过继承`RichParallelSourceFunction`来编写自定义源。

可以从`StreamExecutionEnvironment`使用多个预定义的源。

### 2.1 基于文件

- `readTextFile(path)` 读取文本文件，例如：符合`TextInputFormat`规范的文件，逐行显示，并以字符串形式返回。
- `readFile(fileInputFormat, path)`  根据指定`fileInputFormat`读取一次文件
- `readFile(fileInputFormat, path, watchType, interval, pathFilter, typeInfo)`

### 2.2 基于套接字

- `socketTextStream` 

### 2.3 基于集合

- `fromCollection(Collection)`
- `fromCollection(Iterator, Class)`
- `fromElements(T ...)`
- `fromParallelCollection(SplittableIterator, Class)`
- `generateSequence(from, to)`

### 2.4 用户自定义

- `addSource`

## 3. Data Sink

Data Sinks使用DataSream，并将其转发到文件、套接字、外部系统或打印它们。

- `writeAsText()` / `TextOutputFormat`
- `writeAsCsv(...)` / `CsvOutputFormat`
- `print()` / `printToErr()`
- `writeUsingOutputFormat()` / `FileOutputFormat`
- `writeToSocket`
- `addSink`

## 4. Iterations

迭代流程序实现了一个逐步功能，并将其嵌入到IterativeStream中。 由于DataStream程序可能永远不会完成，因此没有最大迭代次数。 相反，您需要使用拆分转换或过滤器指定流的哪一部分反馈到迭代，以及哪一部分向下游转发。 在这里，我们展示了一个使用过滤器的示例。 首先，我们定义一个IterativeStream:

```java
IterativeStream<Integer> iteration = input.iterate();
```

要关闭迭代并定义迭代尾部，请调用`IterativeStream`的`closeWith(feedbackStream)`方法。 提供给`closeWith`函数的`DataStream`将反馈到迭代头。 一种常见的模式是使用过滤器将反馈的部分流和向前传播的部分分开。 这些过滤器可以例如定义“终止”逻辑，其中允许元素向下游传播而不是被反馈。

## 5. 执行参数

`StreamExecutionEnvironment`包含`ExecutionConfig`，它允许为运行时设置作业特定的配置值。请参考执行配置以获取大多数参数的说明。 这些参数专门与DataStream API有关：

- `setAutoWatermarkInterval(long milliseconds)` 设置自动发射水印间隔，可以使用`long getAutoWatermarkInterval()`获取当前值。

### 5.1 容错

State&Checkpointing描述如何启用和配置Flink  checkpointing机制。

## 6. 控制延迟

默认情况下，元素不会在网络上一对一传输(这会导致不必要的网络通信)，但是会进行缓冲。 缓冲区的大小(实际上是在计算机之间传输的)可以在Flink配置文件中设置。 尽管此方法可以优化吞吐量，但是当传入流不够快时，它可能会导致延迟问题。 要控制吞吐量和延迟，可以在执行环境或各个算子上使用`env.setBufferTimeout(timeoutMillis)`来设置缓冲区填充的最大等待时间。 在此时间之后，即使缓冲区未满，也会自动发送缓冲区。 此超时的默认值为100毫秒。

```java
LocalStreamEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();
env.setBufferTimeout(timeoutMillis);

env.generateSequence(1,10).map(new MyMapper()).setBufferTimeout(timeoutMillis);
```

为了提升吞吐量，设置`setBufferTimeout(-1)`，将删除超时属性，只有在缓冲区满时，才会被刷新。为了减小延迟，将超时设置为接近0的值(例如5或10 ms)。应避免将缓冲区超时设置为0，因为它可能导致严重的性能下降。

## 7. 本地调试

在分布式群集中运行流式程序之前，最好确保已实现的算法按预期工作。 因此，实施数据分析程序通常是检查结果，调试和改进的增量过程。

Flink提供的功能可通过在IDE中支持本地调试，注入测试数据和收集结果数据来大大简化数据分析程序的开发过程。 

### 7.1 Local Execution Environment

LocalStreamEnvironment在与其创建的同一JVM中启动Flink系统，如果从IDE启动LocalEnvironment，则可以在代码中设置断点并轻松调试程序。

一个

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();

DataStream<String> lines = env.addSource(/* some source */);
// build your program

env.execute();
```

### 7.2 Collection Data Sources

Flink提供了由Java集合支持的特殊数据源，以简化测试。 一旦测试了程序，就可以轻松地将源和接收器替换为可读取/写入外部系统的源和接收器。

收集数据源可以按如下方式使用：

```java
final StreamExecutionEnvironment env = StreamExecutionEnvironment.createLocalEnvironment();

// Create a DataStream from a list of elements
DataStream<Integer> myInts = env.fromElements(1, 2, 3, 4, 5);

// Create a DataStream from any Java collection
List<Tuple2<String, Integer>> data = ...
DataStream<Tuple2<String, Integer>> myTuples = env.fromCollection(data);

// Create a DataStream from an Iterator
Iterator<Long> longIt = ...
DataStream<Long> myLongs = env.fromCollection(longIt, Long.class);
```

