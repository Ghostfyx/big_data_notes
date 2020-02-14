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

**例 10-6:用 Scala 进行流式筛选，打印出包含“error”的行**

```scala
// 启动流计算环境StreamingContext并等待它"完成" 
ssc.start()
// 等待作业完成
ssc.awaitTermination()
```

**例 10-7:用 Java 进行流式筛选，打印出包含“error”的行**

```java
// 启动流计算环境StreamingContext并等待它"完成" 
jssc.start();
// 等待作业完成
jssc.awaitTermination();
```

注意：一个StreamingContext只能启动一次，所有只有在配置号所有DStream以及所需要的输出操作后才能启动。启动脚本如下所示：

**例 10-8:在 Linux/Mac 操作系统上运行流计算应用并提供数据**

```sh
$ spark-submit --class com.oreilly.learningsparkexamples.scala.StreamingLogInput \
     $ASSEMBLY_JAR local[4]
     
$ nc localhost 7777 # 使你可以键入输入的行来发送给服务器 <此处是你的输入>
```

接下来会把这个例子加以扩展以处理 Apache 日志文件。如果你需要生成一些假的日志，可以运行本书 Git 仓库中的脚本 ./bin/fakelogs.sh 或者 ./bin/fakelogs.cmd 来把日志发给 7777 端口。

## 10.2 架构与抽象

Spark Streaming使用微批次的架构，把流式计算当作一系列连续的小规模批处理来对待。Spark Streaming从各个输入源中读取数据，并把数据分组为小的批次。新的批次按均匀的时间间隔创建出来，在每个时间区间开始的时候，一个新的批次就创建出来，在该区间内收到的数据都会被添加到这个批次中；在时间区间结束时，批次停止增长。时间区间的大小是由批次间隔这个参数决定的，批次间隔一般设在500毫秒到几秒之间，由应用开发者配置。每一个批次的输入都形成一个RDD，以Spark作业的方式处理并生成其他RDD，处理结果可以以批处理的方式传递给外部系统。高层次架构图如10-1所示：

![](./img/10-1.jpg)

​															**图 10-1:Spark Streaming 的高层次架构**

Spark Streaming的编程抽象是离散流，即DStream，它是一个RDD序列，每个RDD代表数据流中一个时间片内的数据，如图10-2所示：

![](./img/10-2.jpg)

​															**图 10-2:DStream的RDD 序列**

可以对外部数据源创建DStream，也可以对其他DStream进行**转化操作**得到新的DStream，DStream 支持许多第 3 章中所讲到的 RDD 支持的转化操作。另外，DStream 还有**有状态**的转化操作，可以用来聚合不同时间片内的数据。会在后面几节进一步讲解。

运行例10-8，可以从套接字中接收到的数据创建DStream，然后对其应用filter转化操作，这会在内部创建如图10-3所示的RDD。

**例 10-9:运行例 10-8 的日志输出**

```
-------------------------------------------
Time: 1413833674000 ms
-------------------------------------------
71.19.157.174 - - [24/Sep/2014:22:26:12 +0000] "GET /error78978 HTTP/1.1" 404 505
...
-------------------------------------------
Time: 1413833675000 ms
-------------------------------------------
71.19.164.174 - - [24/Sep/2014:22:27:10 +0000] "GET /error78978 HTTP/1.1" 404 505
...
```

![](./img/10-3.jpg)

​									**图 10-3:例 10-4 至例 10-8 中的 DStream 及其转化关系**

筛选过的日志每秒钟被打印一次，这是由于创建StreamingContext时设置的批次间隔为1秒，Spark 用户界面也显示 Spark Streaming 执行了许多小规模作业，如图 10-4 所示（注意：已经完成的步骤中提交时间）。

![](./img/10-4.jpg)

​												**图 10-4:运行流计算作业时的 Spark 应用用户界面**

DStream还支持**输出操作**，比如在示例中使用的 print()。输出操作 和 RDD 的行动操作的概念类似。Spark 在行动操作中将数据写入外部系统中，而 Spark Streaming 的输出操作在每个时间区间中周期性执行，每个批次都生成输出。

Spark Streaming在Spark的驱动程序——工作节点结构中的执行过程如图10-5所示。Spark Streaming为每个输入源启动对应的**接收器**，接收器以任务的形式运行在应用的执行器进程中，从输入源收集数据并保存为RDD。收集到输入数据后会把数据复制到另一个执行器进程来保障容错性，数据保存在内存中，和缓存RDD的方式一致（接收器也可以将数据备份在HDFS上，对于一些输入源：如HDFS，其本身有多个备份，因此Spark Streaming不会再词备份）。驱动器程序中的Streaming Context会周期的奴性Spark作业来处理这些数据，把数据与之前时间区间中的RDD整合。

![](./img/10-5.jpg)

​								**图 10-5:Spark Streaming 在 Spark 各组件中的执行过程**

Spark Streaming对DStream提供的容错性与Spark为RDD所提供的容错性一致：只要输入数据存在，可以使用 RDD 谱系重算出任意状态(比如重新执行处理输入数据的操作)。默认情况下，接收到的数据分别存储在两个节点上，这样 Spark 可以容忍一个工作节点的故障。不过，如果只用谱系图来恢复的话，重算有可能会花很长时间，因为需要处理从 程序启动以来的所有数据。因此，Spark Streaming也提供了**checkPoint检查点机制**。可以把状态阶段性地存储在文件系统中（HDFS、s3等）。一般来说，处理**每5-10个批次数据就保存一次**。在恢复数据时，Spark Streaming只需要回溯到上一个检查点即可。

## 10.3 转化操作

DStream的转化操作可以分为无状态(stateless)和有状态(stateful)两种。

- 在无状态转化操作中，每个批次的处理不依赖于之前批次的数据，第3章和第4章中常见的RD 转化操作，例如 map()、filter()、reduceByKey() 等，都是无状态转化操作。
- 有状态转化操作需要使用之前批次的数据或者是中间结果来计算当前批次的数据。有状态转化操作包括基于滑动窗口的转化操作和追踪状态变化的转化操作。

### 10.3.1 无状态转化操作

无状态转化操作就是把简单的RDD转化操作应用到每个批次上，即转化DStream中的每个RDD。部分无状态转化操作如表10-1所示。**注意**：针对键值对的 DStream 转化操作(比如 reduceByKey())要添加 import StreamingContext._ 才能在 Scala 中使用。和 RDD 一样，在 Java 中需要通过 mapToPair() 创建出一个 JavaPairDStream 才能使用。

​								**表10-1:DStream无状态转化操作的例子(不完整列表)**

| 函数名        | 目的                                                         | 用来操作DStream[T] 的用户自定义函数的 函数签名 |
| ------------- | ------------------------------------------------------------ | ---------------------------------------------- |
| map()         | 对 DStream 中的每个元素应用给 定函数，返回由各元素输出的元素组成的 DStream。 | f: (T) -> U                                    |
| flatMap()     | 对DStream中的每个元素应用给 定函数，返回由各元素输出的迭代器组成的DStream。 | f: T -> Iterable[U]                            |
| filter()      | 返回由给定 DStream 中通过筛选 的元素组成的 DStream。         | f: T -> Boolean                                |
| repartition() | 改变 DStream 的分区数。                                      |                                                |
| reduceByKey() | 将每个批次中键相同的记录归约                                 | f: T, T -> T                                   |
| groupByKey()  | 将每个批次中的记录根据键分组                                 |                                                |

**注意：**这些函数看起来像是作用在整个DStream流上，其实每个DStram内部由RDD序列组成，**无状态状态操作分别作用在每个RDD上**。例如：reduceByKey() 会归约每个时间区间中的数据，但不会归约不同区间之间的数据。

举个例子，在之前的日志处理程序中，可以使用 map() 和 reduceByKey() 在每个时间 区间中对日志根据 IP 地址进行计数，如例 10-10 和例 10-11 所示。

**例 10-10:在 Scala 中对 DStream 使用 map() 和 reduceByKey()**

```scala
// 假设ApacheAccessingLog是用来从Apache日志中解析条目的工具类
val accessLogDStream = logData.map(line => ApacheAccessLog.parseFromLogLine(line)) val ipDStream = accessLogsDStream.map(entry => (entry.getIpAddress(), 1))
val ipCountsDStream = ipDStream.reduceByKey((x, y) => x + y)
```

**例 10-11:在 Java 中对 DStream 使用 map() 和 reduceByKey()**

```java
// 假设ApacheAccessingLog是用来从Apache日志中解析条目的工具类
static final class IpTuple implements PairFunction<ApacheAccessLog, String, Long> {
       public Tuple2<String, Long> call(ApacheAccessLog log) {
         return new Tuple2<>(log.getIpAddress(), 1L);
} }

JavaDStream<ApacheAccessLog> accessLogsDStream = logData.map(new ParseFromLogLine());
JavaPairDStream<String, Long> ipDStream = accessLogsDStream.mapToPair(new IpTuple());
JavaPairDStream<String, Long> ipCountsDStream = ipDStream.reduceByKey(new LongSumReducer());
```

无状态转化操作也能在多个 DStream 间整合数据，不过也是在各个时间区间内。例如，键值对 DStream 拥有和 RDD一样的与连接相关的转化操作，也就是 cogroup()、join()、 leftOuterJoin() 等(见 4.3.3 节)。我们可以在 DStream 上使用这些操作，这样就对每个 批次分别执行了对应的 RDD 操作。

在例 10-12 和例 10-13 中，以 IP 地址为键，把请求计数的数据和传输数据量的数据连接起来。

**例 10-12:在 Scala 中连接两个 DStream**

```scala
val ipBytesDStream =
       accessLogsDStream.map(entry => (entry.getIpAddress(), entry.getContentSize()))
val ipBytesSumDStream = ipBytesDStream.reduceByKey((x, y) => x + y)
val ipBytesRequestCountDStream = ipCountsDStream.join(ipBytesSumDStream)
```

**例 10-13:在 Java 中连接两个 DStream**

```java
JavaPairDStream<String, Long> ipBytesDStream =
       accessLogsDStream.mapToPair(new IpContentTuple());
JavaPairDStream<String, Long> ipBytesSumDStream =
       ipBytesDStream.reduceByKey(new LongSumReducer());
JavaPairDStream<String, Tuple2<Long, Long>> ipBytesRequestCountDStream =
       ipCountsDStream.join(ipBytesSumDStream);
```

还可以像在常规的Spark中一样使用DStream的union() 操作将它和另一个DStream的内容合并起来，也可以使用StreamingContext.union() 来合并多个流。

Spark Streaming提供transform( )高级操作符，可以支持用户自定义直接操作其内部的RDD。这个 transform() 操作允许用户对DStream提供任意一个RDD到RDD的函数。这个函数会在数据流中的每个批次中被调用，生成一 个新的流。transform() 的一个常见应用就是重用为RDD写的批处理代码。例如，如果有extractOutliers() 函数，用来从一个日志记录的 RDD 中提取出异常值的 RDD(可能通过对消息进行一些统计)，你就可以在 transform() 中重用它，如例 10-14 和 例 10-15 所示。

**例 10-14:在 Scala 中对 DStream 使用 transform()**

```scala
val outlierDStream = accessLogsDStream.transform { rdd => extractOutliers(rdd)}
```

**例 10-15:在 Java 中对 DStream 使用 transform()**

```java
JavaPairDStream<String, Long> ipRawDStream = accessLogsDStream.transform(
		new Function<JavaRDD<ApacheAccessLog>, JavaRDD<ApacheAccessLog>>() {
				public JavaPairRDD<ApacheAccessLog> call(JavaRDD<ApacheAccessLog> rdd) {
						return extractOutliers(rdd);
} });
```

也可以通过 StreamingContext.transform或DStream.transformWith(otherStream, 来整合与转化多个 DStream。

### 10.3.2 有状态转化操作

DStream的有状态转化操作时跨时间区跟踪数据的操作，即之前批次数据也被用来在新的批次中计算结果。主要的两种类型是滑动窗口和updateStateByKey( )，前者以一个时间阶段为滑动窗口进行操作，后者则用来跟踪每个键的状态变化。

有状态转化操作需要在StreamingContext中打开检查点机制来确保容错性。会在10.6节中更详细地讨论检查点机制，现在只需要知道可以通过传递一个目录作为参数给 ssc.checkpoint() 来打开它，如例 10-16 所示。

**例 10-16:设置检查点**

```java
ssc.checkpoint("hdfs://...")
```

进行本地开发时，可以使用本地路径(例如：/tmp)取代HDFS。

**基于窗口的转化操作**

基于窗口的转化操作会比一个StreamingContext的批次间隔更长的时间范围内，通过整合多个批次的结果，计算出整个窗口的结果。本节会展示如何使用这种转化操作来跟踪网络服 务器访问日志中的一些信息，比如常见的一些响应代码、内容大小，以及客户端类型。

所有基于窗口的操作都需要两个参数：窗口时长及滑动步长，两者都必须是StreamContext的批次间隔整数倍。窗口时长控制每次计算最近的多少个批次数据，其实就是最近的windowDuration/batchInterval个批次。如果以10秒为批次间隔的源DStream，要创建最近30秒的时间窗口(即最近 3 个批次)，就应当把 windowDuration设为30 秒。而滑动步长的默认值与批次间隔相等，用来控制对新的 DStream 进行计算的间隔。如果源 DStream 批次间隔为 10 秒，并且我们只希望每两个批次计算一次窗口结果， 就应该把滑动步长设置为20秒。图 10-6 展示了一个例子。

![](./img/10-6.jpg)

**图 10-6:一个基于窗口的流数据，窗口时长为 3 个批次，滑动步长为 2 个批次;每隔 2 个批次就对 前 3 个批次的数据进行一次计算**

对DStream可以用的最简单的窗口操作是window( )，它返回一个新的DStream来表示所请求的窗口操作的结果数据，即window( )生成的DStream中的每个RDD会包含多个批次中的数据，可以对这些数据进行count( )，transform( )等操作，见例10-17与10-18。

**例 10-17:如何在 Scala 中使用 window() 对窗口进行计数**

```scala
val accessLogsWindow = accessLogsDStream.window(Seconds(30), Seconds(10))
val windowCounts = accessLogsWindow.count()
```

**例 10-18:如何在 Java 中使用 window() 对窗口进行计数**

```java
JavaDStream<ApacheAccessLog> accessLogsWindow = accessLogsDStream.window(
         Durations.seconds(30), Durations.seconds(10));
JavaDStream<Integer> windowCounts = accessLogsWindow.count();
```

Spark Streaming还提供了一些其他的窗口操作。reduceByWindow() 和 reduceByKeyAndWindow() 让我们可以对每个窗口更高效地进行归约操作。它们接收一个归约函数，在整个窗口上执。此外，还有一种特殊的形式，通过只考虑新进入窗口的数据和离开窗口的数据，让Spark增量的计算归约结果，这种特殊形式需要提供归约函数的一个**逆函数**，比 如 + 对应的逆函数为 -。对于较大的窗口，提供逆函数可以大大提高执行效率(见图 10-7)。

![](./img/10-7.jpg)

在日志处理的例子中，可以使用这两个函数来更高效地对每个 IP 地址访问量进行计数，如例 10-19 和例 10-20 所示。

**例 10-19:Scala 版本的每个 IP 地址的访问量计数**

```scala
val ipDStream = accessLogsDStream.map(logEntry => (logEntry.getIpAddress(), 1))
val ipCountDStream = ipDStream.reduceByKeyAndWindow(
    {(x, y) => x + y}, // 加上新进入窗口的批次中的元素
    {(x, y) => x - y}, // 移除离开窗口的老批次中的元素
    Seconds(30), // 窗口时长
    Seconds(10) // 滑动步长
)
```

**例 10-20:Java 版本的每个 IP 地址的访问量计数**

```java
class ExtractIp extends PairFunction<ApacheAccessLog, String, Long> {
      public Tuple2<String, Long> call(ApacheAccessLog entry) {
        return new Tuple2(entry.getIpAddress(), 1L);
      }
    }
    class AddLongs extends Function2<Long, Long, Long>() {
      public Long call(Long v1, Long v2) { return v1 + v2; }
    }
    class SubtractLongs extends Function2<Long, Long, Long>() {
      public Long call(Long v1, Long v2) { return v1 - v2; }
}
JavaPairDStream<String, Long> ipAddressPairDStream = accessLogsDStream.mapToPair(
      new ExtractIp());
JavaPairDStream<String, Long> ipCountDStream = ipAddressPairDStream. reduceByKeyAndWindow(
      new AddLongs(), // 加上新进入窗口的批次中的元素
      new SubtractLongs() // 移除离开窗口的老批次中的元素 
      Durations.seconds(30), // 窗口时长 
      Durations.seconds(10)); // 滑动步长
```

**UpdateStateByKey转化操作**

有时，需要在DStream中跨批次维护状态，例如：跟踪用户访问网站的会话。updateStateByKey()提供了对一个状态变量的访问，用于键值对形式的 DStream。给定一个< Key, Value>构成的DStream，并传递如何根据新的事件更新每个键对应状态的函数，最终返回一个新的DStream，其内部数据为(键，状态)。例如：在网络服务器日志中，事件可能是对网站的访问，此时键是用户的ID，使用updateStateByKey( )可以跟踪每个用户最近访问的10个页面。这个列表就是“状态”对 象，我们会在每个事件到来时更新这个状态。

使用updateStateByKey( )，提供一个update(events, oldState)函数，接收与Key相关的事件以及该Key之前对应的状态，返回这个键对应的新状态。函数签名如下：

- event：当前批次中接收到的事件列表（可能为空）。
- oldState：是键之前的状态对象，存放在Option内，如果一个Key之前没有状态，这个值可以为空，
- newState：由函数返回，也以 Option 形式存在；可以返回一个空的 Option 来表示要删除该状态。

updateStateByKey( )的结果是一个新的DStream，其内部RDD序列是由每个时间区间对应的(键，状态)组成。

举个简单的例子，使用 updateStateByKey() 来跟踪日志消息中各 HTTP 响应代码的计数。这里的键是响应代码，状态是代表各响应代码计数的整数，事件则是页面访问。请注意，跟之 前的窗口例子不同的是，例 10-23 和例 10-24 会进行自程序启动开始就“无限增长”的计数。

**例 10-23:在 Scala 中使用 updateStateByKey() 运行响应代码的计数**

```scala
def updateRunningSum(values: Seq[Long], state: Option[Long]) = {
       Some(state.getOrElse(0L) + values.size)
}
val responseCodeDStream = accessLogsDStream.map(log => (log.getResponseCode(), 1L)) 
val responseCodeCountDStream = responseCodeDStream.updateStateByKey(updateRunningSum _)
```

**例 10-24:在 Java 中使用 updateStateByKey() 运行响应代码的计数**

```java
class UpdateRunningSum implements Function2<List<Long>,
         Optional<Long>, Optional<Long>> {
       public Optional<Long> call(List<Long> nums, Optional<Long> current) {
         long sum = current.or(0L);
         return Optional.of(sum + nums.size());
} };
JavaPairDStream<Integer, Long> responseCodeCountDStream = accessLogsDStream.mapToPair( new PairFunction<ApacheAccessLog, Integer, Long>() {
           public Tuple2<Integer, Long> call(ApacheAccessLog log) {
             return new Tuple2(log.getResponseCode(), 1L);
         }})
       .updateStateByKey(new UpdateRunningSum());
```

## 10.4 输出操作

输出操作指定了对流数据经转化操作得到的数据所要执行的操作，例如：结果推入外部数据库或输出到屏幕。输出操作与RDD中的惰性求值类似，如果DStream及其派生出的DStream都没有被执行输出操作，则DStream就不会被求值；如果StreamingContext中没有设定输出操作，整个context就都不会启动。

print( )是常用的调试性输出操作，会在每个批次中抓取DStream的前10个元素打印。Spark Streaming对DStream有与Spark类似的save( )操作，它们接收一个目录作为参数存储文件，还支持通过可选参数来设置文件的后缀名，每个批次的结果被保存在给定目录的子目录里，且文件名还有时间和后缀名，如例 10-25 所示。

例 10-25:在 Scala 中将 DStream 保存为文本文件 

```scala
ipAddressRequestCount.saveAsTextFiles("outputDir", "txt")
```

还有一个更为通用的 **saveAsHadoopFiles**() 函数，接收一个 Hadoop 输出格式作为参数。例如，Spark Streaming 没有内建的 saveAsSequenceFile() 函数，但是可以使用例 10-26 和例 10-27 中的方法来保存 SequenceFile 文件。

**例 10-26:在 Scala 中将 DStream 保存为 SequenceFile**

```scala
val writableIpAddressRequestCount = ipAddressRequestCount.map {
       (ip, count) => (new Text(ip), new LongWritable(count)) }
writableIpAddressRequestCount.saveAsHadoopFiles[
       SequenceFileOutputFormat[Text, LongWritable]]("outputDir", "txt")
```

**例 10-27:在 Java 中将 DStream 保存为 SequenceFile**

```java
JavaPairDStream<Text, LongWritable> writableDStream = ipDStream.mapToPair(
       new PairFunction<Tuple2<String, Long>, Text, LongWritable>() {
         public Tuple2<Text, LongWritable> call(Tuple2<String, Long> e) {
           return new Tuple2(new Text(e._1()), new LongWritable(e._2()));
       }});
class OutFormat extends SequenceFileOutputFormat<Text, LongWritable> {};
     writableDStream.saveAsHadoopFiles(
       "outputDir", "txt", Text.class, LongWritable.class, OutFormat.class);
```

最后，还有一个通用的输出操作foreachRDD( )，它用来对DStream中的RDD运行任意计算，这和transform有些类似，都可以让我们访问任意RDD，在 foreachRDD() 中，可以重用在Spark中实现的所有行动操作。比如，常见的用例之一是把数据写到诸如 MySQL 的外部数据库中。对于这种操作，Spark 没有提供对应的 saveAs() 函数，但可以使 用 RDD 的 eachPartition() 方法来把它写出去。为了方便，foreachRDD() 也可以提供给当前批次的时间，允许把不同时间的输出结果存到不同的位置。参见例 10-28。

**例 10-28:在 Scala 中使用 foreachRDD() 将数据存储到外部系统中**

```scala
ipAddressRequestCount.foreachRDD { rdd =>
  	rdd.foreachPartition { partition =>
// 打开到存储系统的连接(比如一个数据库的连接) 
      partition.foreach { item =>
// 使用连接把item存到系统中 
      }
// 关闭连接 
    }
}
```

## 10.5 输入源

Spark Streaming原生支持一些不同的数据源。一些“核心”数据源已经被打包到 Spark Streaming 的 Maven 工件中，而其他的一些则可以通过 spark-streaming-kafka 等附加工件 获取。

假如在设计一个新的应用，建议从使用HDFS和Kafka这种简单的输入源开始。

### 10.5.1 核心数据源

所有从核心数据源创建DStream的方法都位于StreamingContext中，前面小节已经使用过一个：**套接字**(ssc.socketTextStream("localhost", 7777))，下面讨论文件和Akka actor。

**1.文件流**

因为Spark支持从任意Hadoop兼容的文件系统中读取数据，所以Spark Streaming也支持从任意Hadoop兼容的文件系统目录中的文件创建数据流。由于支持多种后端，这种方 式广为使用，尤其是对于像日志这样始终要复制到 HDFS 上的数据。尤其是对于像日志这样始终要复制到HDFS上的数据，要让Spark Streaming来处理数据，需要为目录名字提供**统一的日期格式**，文件也必须**原子化创建**。所谓原子化创建指的是：文件创建与数据写入操作一次完成，如果Spark Streaming要处理文件时，更多数据出现了，Spark Streaming会无法注意到新增加的数据，因此原子化操作对Spark Streaming读取文件流数据非常重要。可以修改例 10-4 和例 10-5 来处理新出现在一个目录下的日 志文件，如例 10-29 和例 10-30 所示。

**例 10-29:用 Scala 读取目录中的文本文件流**

```scala
val logData = ssc.textFileStream(logDirectory)
```

**例 10-30:用 Java 读取目录中的文本文件流**

```java
 JavaDStream<String> logData = jssc.textFileStream(logsDirectory);
```

除了文本数据，也可以读入任意Hadoop输入格式，与5.2.6节所讲一样，只需要将Key、Value以及InputFormat类提供给Spark Streaming即可。例如，如果先前已经有了一 个流处理作业来处理日志，并已经将得到的每个时间区间内传输的数据分别存储成了一个 SequenceFile，就可以如例 10-31 中所示的那样来读取数据。

**例 10-31:用 Scala 读取目录中的 SequenceFile 流**

```scala
ssc.fileStream[LongWritable, IntWritable,
         SequenceFileInputFormat[LongWritable, IntWritable]](inputDirectory).map {
         case (x, y) => (x.get(), y.get())
}
```

**2.Akka actor流**

另一个核心数据源接收器是 actorStream，它可以把 Akka actor(http://akka.io/)作为数据 流的源。要创建出一个 actor 流，需要创建一个 Akka actor，然后实现 org.apache.spark. streaming.receiver.ActorHelper 接口。要把输入数据从 actor 复制到 Spark Streaming 中， 需要在收到新数据时调用 actor 的 store() 函数。Akka actor 流不是很常见，所以不会在此对其进行深入探究。

### 10.5.2 附加数据源

除核心数据源外，还可以使用附加数据源从一些知名数据获取系统中接收数据，这些接收器都作为 Spark Streaming 的组件进行独立打包了，仍然是 Spark 的一部分，不过需要在构建文件中添加额外的包才能使用它们。现有的接收器包括 Twitter、Apache Kafka、Amazon Kinesis、Apache Flume，以及 ZeroMQ。可以通过添加与 Spark 版本匹配的Maven工件 spark-streaming-[projectname]_2.10 来引入这些附加接收器。

**1. Apache Kafka**

Apache Kafka因其速度和弹性成为一个流行的输入源。使用kafka原生支持，可以轻松处理许多主题的消息，在工程中需要引入Maven工件spark- streaming-kafka_2.10来使用它。包内提供KafkaUtils对象可以在StreamingContext和JavaStreamingContext中以Kafka消息创建出DStream。由于KafkaUtils可以订阅多个主题，因此它创建出的DStream由成对的主题和消息组成。要创建出一个数据流，需要使用**StreamingContext实例、一个逗号隔开的ZooKeeper主机列表字符串、消费者组的名字（唯一名字），以及一个从主题到针对这个主题的接收器的映射表来调用createStream方法**。如例10-32和例10-33所示：

**例 10-32:在 Scala 中用 Apache Kafka 订阅 Panda 主题**

```scala
import org.apache.spark.streaming.kafka._
...
// 创建一个从主题到接收器线程数的映射表
val topics = List(("pandas", 1), ("logs", 1)).toMap
val topicLines = KafkaUtils.createStream(ssc, zkQuorum, group, topics)
StreamingLogInput.processLines(topicLines.map(_._2))
```

**例 10-33:在 Java 中用 Apache Kafka 订阅 Panda 主题**

```java
import org.apache.spark.streaming.kafka.*;
...
// 创建一个从主题到接收器线程数的映射表
Map<String, Integer> topics = new HashMap<String, Integer>(); 
topics.put("pandas", 1);
topics.put("logs", 1);
JavaPairDStream<String, String> input =
       KafkaUtils.createStream(jssc, zkQuorum, group, topics);
input.print();
```

**2. Apache Flume**

Spark提供两种不同的接收器：推式接收器和拉链接收器来使用Apache Flume，见图10-8。

![](./img/10-8.jpg)

​																	**图 10-8:Flume 接收器选项**

- 推式接收器

	推式接收器以Avro数据池的方式工作，由Flume向其中推数据。

- 拉式接收器

	该接收器可以从自定义的中间数据池中拉数据，而其他进程可以使用Flume把数据推进该中间数据池。

两种方式都需要重新配置Flume，并在某个节点配置的端口上运行接收器（不是已有的Spark或者Flume使用的端口）。要使用其中任何一种方法，都需要在工程中引入 Maven 工件 spark-streaming-flume_2.10。

**3. 推式接收数据**

推式接收器的方法设置起来很容易，但是它不使用事务来接收数据。在这种方式中，接收数据已Avro数据池的方式工作，需要配置Flume来把数据发到Avro数据池（例10-34）。提供FlumeUtils对象会把接收器配置在一个特定工作节点的主机名和端口号上（例10-35与10-36）。这些设置必须和 Flume 配置相匹配。

**例 10-34:Flume 对 Avro 池的配置**

```xml
a1.sinks = avroSink
a1.sinks.avroSink.type = avro
a1.sinks.avroSink.channel = memoryChannel
a1.sinks.avroSink.hostname = receiver-hostname
a1.sinks.avroSink.port = port-used-for-avro-sink-not-spark-port
```

**例 10-35:Scala 中的 FlumeUtils 代理**

```scala
 val events = FlumeUtils.createStream(ssc, receiverHostname, receiverPort)
```

**例 10-36:Java 中的 FlumeUtils 代理**

```java
JavaDStream<SparkFlumeEvent> events = FlumeUtils.createStream(ssc, 										   		 																						receiverHostname,receiverPort)
```

这种方式没有事务支持，这会增加运行接收器的工作节点发送错误时丢失数据的概率。不仅如此，如果运行接收器的工作节点发生故障，系统会尝试从 另一个位置启动接收器，这时需要重新配置 Flume 才能将数据发给新的工作节点。这样配 置会比较麻烦。

**4. 拉式接收器**

较新的方式是拉式接收器（在Spark1.1中引入），它设置了一个专用的Flume数据池供Spark Streaming读取，并让接收器主动从数据池中拉取数据，这种方式的优点在于弹性较好，Spark Streaming通过事务从数据池中读取并复制数据，收到事务完成的通知前，这些数据还保留在数据池中。

我们需要先把自定义数据池配置为Flume第三方插件，安装插件的最新方法请参考 Flume 文档的相关部分(https://flume.apache.org/FlumeUserGuide.html#installing-third-party- plugins)。由于插件是用 Scala 写的，因此需要把插件本身以及 Scala 库都添加到 Flume 插件中。Spark 1.1 中对应的 Maven 索引如例 10-37 所示。

**例 10-37:Flume 数据池的 Maven 索引**

```xml
     groupId = org.apache.spark
     artifactId = spark-streaming-flume-sink_2.10
     version = 1.2.0

     groupId = org.scala-lang
     artifactId = scala-library
     version = 2.10.4
```

当把自定义Flume数据池添加到一个节点之上后，就需要配置Flume来把数据推送到这个数据池中，如例10-38所示：

**例 10-38:Flume 对自定义数据池的配置**

```xml
a1.sinks = spark
a1.sinks.spark.type = org.apache.spark.streaming.flume.sink.SparkSink
a1.sinks.spark.hostname = receiver-hostname
a1.sinks.spark.port = port-used-for-sync-not-spark-port
a1.sinks.spark.channel = memoryChannel
```

等到数据已经在数据池缓存起来，就可以使用FlumeUtils来读取数据了，如例10-39和10-40所示：

例 10-39:在 Scala 中使用 FlumeUtils 读取自定义数据池

```scala
 val events = FlumeUtils.createPollingStream(ssc, receiverHostname, receiverPort)
```

例 10-40:在 Java 中使用 FlumeUtils 读取自定义数据池 

```java
JavaDStream<SparkFlumeEvent> events = FlumeUtils.createPollingStream(ssc,
       receiverHostname, receiverPort)
```

DStream是由SparkFlumeEvent组成的，可以通过event方法访问下层的AvroFlumeEvent。

### 10.5.3 多数据源与集群规模

可以使用类似union( )这样的操作将多个DStream合并，通过这些操作符，可以把多个输入的DStream合并起来。这样的做法有两个好处：1. 使用多个接收器对于提高聚合操作的数据获取的吞吐量非常必要，如果只有一个接收器，会成为系统的瓶颈；2. 现实场景下，有时需要使用不同的接收器从不同的数据源接收各种数据，然后join或者cogroup进行整合。

每个接收器都以 Spark 执行器程序中一个长期运行的任务的形式运行，因此会占据分配给应用的CPU核心。此外，还需要有可用的CPU核心来处理数据。如果要运行多个接收器，则集群CPU核心数必须大于结束器个数。

## 10.6 24/7不间断运行

Spark Streaming的一大优势在于它提供了强大的容错性保障。只要输入数据存储在可靠的系统中，Spark Streaming就可以根据输入计算出正确的结果，提供“精确一次”执行的语义(就好像所有的数据都是在没有任何节点失败的情况下处理的一样)，即使是工作节点或者驱动器程序发生了失败。

要不间断运行Spark Streaming应用，需要一些特别的配置：

1. 是设置好诸如HDFS或Amazon S3等可靠存储系统中的**检查点机制** ；
2. 考虑驱动器程序的容错性以及对不可靠输入源的处理。

### 10.6.1 检查点机制

检查点机制是Spark Streaming中用来保障容错性的主要机制，它可以使Spark Streaming阶段性的把数据存储在HDFS或者Amazon S3这样的可靠存储系统中，以供恢复使用。具体来说，检查点机制主要为以下两个目的：

- 控制发生失败时需要重算的状态数，由于Spark Streaming可以通过转化图的谱系图来重算状态，检查点机制可以控制需要在转化图中回溯多远。
- 提供驱动器容错。如果Spark Streaming的驱动器程序崩溃了，可以重启驱动器程序并让驱动器程序从检查点恢复，这样Spark Streaming就可以读取之前运行的程序处理数据的进度，并从那里继续。

出于这些原因，检查点机制对于任何生产环境中的流计算应用都至关重要。你可以通过向 ssc.checkpoint() 方法传递一个路径参数(HDFS、S3 或者本地路径均可)来配置检查点 机制，如例 10-42 所示。

例 10-42:配置检查点

```
 ssc.checkpoint("hdfs://...")
```

注意：即使在本地模式下，如果一个有状态操作而没有打开检查点机制，Spark Streaming也会给出提示。

### 10.6.2 驱动器容错

驱动器程序的容错需要使用特殊的方式创建StreamingContext，与之前使用的new StreamingContext不同，应该使用StreamingContext.getOrCreate( )函数，并把检查点目录传给StreamingContext。如例 10-43 和例 10-44 所示。

**例 10-43:用 Scala 配置一个可以从错误中恢复的驱动器程序**

```scala
def createStreamingContext() = {
       ...
		val sc = new SparkContext(conf)
		// 以1秒作为批次大小创建StreamingContext
		val ssc = new StreamingContext(sc, Seconds(1)) ssc.checkpoint(checkpointDir)
}
...
val ssc = StreamingContext.getOrCreate(checkpointDir, createStreamingContext _)
```

**例 10-44:用 Java 配置一个可以从错误中恢复的驱动器程序**

```java
JavaStreamingContextFactory fact = new JavaStreamingContextFactory() {
       public JavaStreamingContext call() {
					...
					JavaSparkContext sc = new JavaSparkContext(conf);
          // 以1秒作为批次大小创建StreamingContext
          JavaStreamingContext jssc = new JavaStreamingContext(sc, Durations.seconds(1)); 
         	jssc.checkpoint(checkpointDir);
          return jssc;
       }};
JavaStreamingContext jssc = JavaStreamingContext.getOrCreate(checkpointDir, fact);
```

当上面代码第一次运行时，假设检查点目录还不存在，那么StreamingContext会调用工厂函数(在 Scala 中为 createStreamingContext()，在 Java 中为 JavaStreamingContextFactory()) 时把目录创建出来。在驱动程序失败后，如果重启驱动程序并再次执行代码，getOrCreate方法会重新冲检查点目录中初始化出StreamingContext，然后继续处理。

除了用 getOrCreate() 来实现初始化代码以外，还需要编写在驱动器程序崩溃时重启驱动器进程的代码。在大多数集群管理器中，Spark 不会在驱动器程序崩溃时自动重启驱动 器进程，所以需要使用诸如**monit**这样的工具来监视驱动器进程并进行重启。最佳的实现方式往往取决于具体环境。Spark 在独立集群管理器中提供了更丰富的支持，可以在提交驱动器程序时使用 --supervise 标记来让 Spark 重启失败的驱动器程序。还要传递 --deploy-mode cluster 参数来确保驱动器程序在集群中运行，而不是在本地机器上运Spark Streaming ，如例 10-45 所示。
 **例 10-45:使用监管模式启动驱动器程序**

```
./bin/spark-submit --deploy-mode cluster --supervise --master spark://... App.jar
```

在使用这个选项时，如果希望Spark独立模式集群的主节点也是容错的，就可以通过 ZooKeeper 来配置主节点的容错性，

### 10.6.3 工作节点容错

为了应对工作节点失败的问题，Spark Streaming使用与Spark容错机制相同的房吧，所有从外部数据源收到的数据都在多个工作节点上备份。所有从备份数据转化操作的过程 中创建出来的 RDD 都能容忍一个工作节点的失败，因为根据 RDD 谱系图，系统可以把丢失的数据从幸存的输入数据备份中重算出来。

### 10.6.4 接收器容错

运行接收器的工作节点上容错也是很重要的，如果接收器节点发生错误，Spark Streaming会在集群其他节点上重启失败的接收器。这种情况下会不会导致数据丢失取决于数据源行为（数据源是否会重发数据）以及接收器的实现（接收器是否会向数据源确认收到的数据）。以Flume为例：两种接收器的主要区别在于数据丢失时的保障。

- 拉式接收器：接收器从数据源拉取数据，Spark只会在数据已经在集群中备份时，才会从数据池中移除数据；
- 推式接收器：如果接收器在数据 备份之前失败，一些数据可能就会丢失。

总的来说，对于任意一个接收器，必须同时考虑上游数据源的容错性(是否支持事务)来确保零数据丢失。

接收器提供以下保证，确保数据无丢失：

- 从可靠数据源读取的数据(比如通过 StreamingContext.hadoopFiles 读取的)，其底层文件系统会有备份，Spark Streaming会记住哪些数据存放到了检查点中，并在应用崩溃后从检查点处继续执行。
- 对于像 Kafka、推式 Flume、Twitter 这样的不可靠数据源，Spark会把输入的数据复制到其他节点上，但是如果接收器任务崩溃，Spark还是会丢失数据。在Spark1.2中，收到的数据被记录到诸如HDFS这样的可靠的文件系统中，这样即使驱动器程序重启也不会导致数据丢失。

综上所述，确保所有数据都被处理的最佳方式是使用可靠的数据源（例如：HDFS，拉式Flume等）。如果还需要在批处理作业中处理这些数据，使用可靠数据源是最佳方式，因为这种方式确保了批处理作业和流计算作业能读取到相同的数据，因而可以得到相同的 结果。

### 10.6.5 处理保证

由于 Spark Streaming 工作节点的容错保障，Spark Streaming 可以为所有的转化操作提供 “精确一次”执行的语义，即使一个工作节点在处理部分数据时发生失败，最终的转化结果(即转化操作得到的 RDD)仍然与数据只被处理一次得到的结果一样。

然而，当把转化操作得到的结果使用输出操作推入外部系统时，写入结果的任务可能因为故障而执行多次，一些数据可能被写入多次，由于引入了外部系统，因此需要针对个系统的代码来处理多次写入情况。

- 使用事务操作来写入外部系统，即原子化的将一个RDD分区一次写入，例如Spark Streaming的saveAs...File会在一个文件写完时自动将其原子化地移动到最终位置上，以此确保每个输出文件只存在一份。
- 设计幂等的更新操作，即多次运行同一更新操作仍生成相同的结果。

## 10.7 Streaming 用户界面

Spark Streaming 提供了一个特殊的用户界面，可以让我们查看应用在干什么。这个界 面在常规的 Spark 用户界面(一般为 http://:4040)上的 Streaming 标签页里。图 10-9 是 Streaming 用户界面的一个截屏。

Streaming用户界面展示了批处理和接收器的统计信息。在所举的例子中，有一个网路接收器，借此可以看到消息处理的速率。如果处理的速率较慢，就可以看到每个接收器可以处理多少条数据，也可以看到接收器是否发生了故障。批处理的统计信息则呈现批处理已经占用的时长，以及调度作业时的延迟情况。如果集群遇到了资源竞争，那么调度的延迟会增长。

![](./img/10-9.jpg)

## 10.8 性能考量

除了已经针对一般的 Spark 应用讨论过的性能考量以外，Spark Streaming 应用还有一 些特殊的调优选项。

### 10.8.1 批次和窗口大小

最常见的问题是Spark Streaming可以使用的最小批次间隔是多少？总的来说，**500毫秒**已经被证实为对许多应用而言是比较好的最小批次大小。寻找最小批次大小的最佳实践是从一个比较大的批次大小(10秒左右)开始，不断使用更小的批次大小。如果 Streaming 用户界面中显示的处理时间保持不变，就可以进一步减小批次大小。如果处理时间开始增 加，可能已经达到了应用的极限。

相似的，对与窗口操作、计算结果的间隔(滑动步长)对于性能也有巨大的影响。 当计算代价巨大并成为系统瓶颈时，就应该考虑提高滑动步长了。

### 10.8.2 并行度

减少批处理所消耗时间的常见方式还有提高并行度，以下有三种方式可以提高并行度：

- 增加接收器数目

	如果有时记录太多导致单台机器来不及读入并分化的话，接收器会成为系统的瓶颈。需要通过创建多个DStream(这样会创建多个接收器)来增加接收器数目，然后使用union来把数据合并为一个数据源。

- 将接收到的数据显式地重新分区

	如果接收器的数目无法再增加，可以通过使用DStream.repartition来显式的重新分区输入流来重新分配接收到的数据。

- 提高聚合计算的并行度

	对于像reduceByKey( )这样的操作，可以在第二个参数中指定并行度。

### 10.8.3 垃圾回收与内存使用

Java的垃圾回收机制会引起的不可预测的长暂停。可以通过打开Java的并发标志—清除收集器(Concurrent Mark-Sweep garbage collector)来减少GC引起的不可预测的长暂停，并发标志—清除收集器总体上会消耗更多的资源，但是会减少暂停次数。

可以通过在配置参数 spark.executor.extraJavaOptions 中添加 -XX:+UseConcMarkSweepGC 来控制选择并发标志—清除收集器。例 10-46 展示了使用 spark-submit 时的配置方法。

**例 10-46:打开并发标志—清除收集器**

```sh
spark-submit --conf spark.executor.extraJavaOptions=-XX:+UseConcMarkSweepGC App.jar
```

除了使用较少引发暂停的GC回收器，还可以通过减轻GC的压力来大幅改善性能。把RDD以序列化的格式缓存，而不使用原生对象，也可以减轻 GC 的压力，这也是为 什么默认情况下 Spark Streaming 生成的 RDD 都以序列化后的格式存储。使用 Kryo 序列化 工具可以进一步减少缓存在内存中的数据所需要的内存大小。

Spark也允许控制缓存下来的RDD以怎样的策略从缓存中移除。默认情况，Spark使用LRU，如果设置了 spark.cleaner.ttl，Spark 也会显式移除超出给定时间范围 的老 RDD。主动从缓存中移除不大可能再用到的 RDD，可以减轻 GC 的压力。

## 10.9 总结

本章学习了如何使用DStream操作流数据，由于DStream是由RDD组成的，前面章节的知识也适用于流式计算与实时应用。