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