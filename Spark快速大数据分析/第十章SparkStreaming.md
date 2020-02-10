# 第十章 Spark Streaming

许多应用需要即时处理收到的数据，例如用来实时追踪页面访问统计的应用、训练机器学习模型的应用，还有自动检测异常的应用。Spark Streaming是Spark为这些应用而设计的模型。它允许用户使用一套和批处理非常接近的 API 来编写流式计算应用，这样就可以大量重用批处理应用的技术甚至代码。

和Spark基于RDD的概念相似，Spark Streaming使用离散化（discretized stream）作为抽象表示，叫做DStream，DStream是随着时间推移而收到的数据序列。在内部，每个时间区间收到的数据都作为RDD存在，而DStream是由这些RDD组成的序列。DStream 可以从各种输入源创建，比如 Flume、Kafka 或者 HDFS。创 建出来的 DStream 支持两种操作，一种是转化操作(transformation)，会生成一个新的 DStream；另一种是输出操作(output operation)，可以把数据写入外部系统中。DStream 提供了许多与 RDD 所支持的操作相类似的操作支持，还增加了与时间相关的新操作，比如滑动窗口。

与批处理程序不同，Spark Streaming应用需要进行额外配置来保证24/7不间断工作，本章会讨论检查点机制，把数据存储在可靠文件系统（比如HDFS）上的机制。这是 Spark Streaming 用来实现不间断工作的主要方式。此外，还会讲到在遇 到失败时如何重启应用，以及如何把应用设置为自动重启模式。

## 10.1 简单例子

我们从一台服务器的7777端口上收到一个以换行符分隔的多行文本，从中筛选出包含单词error的行，并打印出来。

Spark Streaming 程序最好以使用 Maven 或者 sbt 编译出来的独立应用的形式运行。Spark Streaming 虽然是 Spark 的一部分，它在 Maven 中也以独立工件的形式提供，你也需要在 工程中添加一些额外的 import 声明，如例 10-1 至例 10-3 所示。

**例 10-1:Spark Streaming 的 Maven 索引**

```
     groupId = org.apache.spark
     artifactId = spark-streaming_2.10
     version = 1.2.0
```

**例 10-2:Scala 流计算 import 声明**

```scala
     import org.apache.spark.streaming.StreamingContext
     import org.apache.spark.streaming.StreamingContext._
     import org.apache.spark.streaming.dstream.DStream
     import org.apache.spark.streaming.Duration
     import org.apache.spark.streaming.Seconds
```

**例 10-3:Java 流计算 import 声明**

```java
     import org.apache.spark.streaming.api.java.JavaStreamingContext;
     import org.apache.spark.streaming.api.java.JavaDStream;
     import org.apache.spark.streaming.api.java.JavaPairDStream;
     import org.apache.spark.streaming.Duration;
     import org.apache.spark.streaming.Durations;
```

第一步创建StreamingContext，它是流式计算的主要入口，StreamingContxt在底层创建出SparkContext，StreamingContext构造函数还接收指定处理时长处理一次新数据的批次间隔（batch interval）作为输入。

第二步调用socketTextStream() 来创建出基于本地 7777 端口上收到的文本数据的 DStream。

第三步把DStream通过 filter() 进行转化，只得到包含“error”的行。

第四步使用输出操作 print() 把一些筛选出来的行打印出来。(如例 10-4 和例 10-5 所示。)。

**例 10-4:用 Scala 进行流式筛选，打印出包含“error”的行**

```scala
// 从SparkConf创建StreamingContext并指定1秒钟的批处理大小 
val ssc = new StreamingContext(conf, Seconds(1))
// 连接到本地机器7777端口上后，使用收到的数据创建DStream 
val lines = ssc.socketTextStream("localhost", 7777)
// 从DStream中筛选出包含字符串"error"的行
val errorLines = lines.filter(_.contains("error")) 
// 打印出有"error"的行
errorLines.print()
```

**例 10-5:用 Java 进行流式筛选，打印出包含“error”的行**

```java
// 从SparkConf创建StreamingContext并指定1秒钟的批处理大小
JavaStreamingContext jssc = new JavaStreamingContext(conf, Durations.seconds(1)); 
// 以端口7777作为输入来源创建DStream
JavaDStream<String> lines = jssc.socketTextStream("localhost", 7777);
// 从DStream中筛选出包含字符串"error"的行
JavaDStream<String> errorLines = lines.filter(new Function<String, Boolean>() {
       public Boolean call(String line) {
         return line.contains("error");
}});
// 打印出有"error"的行 
errorLines.print();
```

只要是设定好了要进行的计算，系统收到数据时计算就会开始。要开始接收数据，必须显式的调用StreamingContext的start( )方法，Spark Streaming会把Spark作业不断交给SparkContext调度执行。执行会在另一个线程中进行，所以需要调用 awaitTermination 来等待流计算完成，来防止应用退出。(见例 10-6 和例 10-7。)

例 10-6:用 Scala 进行流式筛选，打印出包含“error”的行

```scala
// 启动流计算环境StreamingContext并等待它"完成" 
ssc.start()
// 等待作业完成
ssc.awaitTermination()
```

例 10-7:用 Java 进行流式筛选，打印出包含“error”的行

```java
// 启动流计算环境StreamingContext并等待它"完成" 
jssc.start();
// 等待作业完成
jssc.awaitTermination();
```

注意：一个StreamingContext只能启动一次，所有只有在配置号所有DStream以及所需要的输出操作后才能启动。启动脚本如下所示：

```sh
$ spark-submit --class com.oreilly.learningsparkexamples.scala.StreamingLogInput \
     $ASSEMBLY_JAR local[4]
     
$ nc localhost 7777 # 使你可以键入输入的行来发送给服务器 <此处是你的输入>
```

接下来会把这个例子加以扩展以处理 Apache 日志文件。如果你需要生成一些假的日志，可以运行本书 Git 仓库中的脚本 ./bin/fakelogs.sh 或者 ./bin/fakelogs.cmd 来把日志发给 7777 端口。

## 10.2 架构与抽象

Spark Streaming使用微批次的架构，把流式计算当作一系列连续的小规模批处理来对待。Spark Streaming从各个输入源中读取数据，并把数据分组为小的批次。新的批次按均匀的时间jiane