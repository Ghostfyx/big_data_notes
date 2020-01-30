# 第六章 Spark编程进阶

## 6.1 简介

本章回介绍Spark两种类型的共享变量：累加器(accumulator)与广播变量(brodcast variable)。累加器用来对信息进行聚合，广播变量用于高效分发较大的对象。在已有的 RDD 转化操作的基础上，我们为类似查询数据库这样需要很大配置代价的任务引入了批操作。

本章会使用业余无线电操作者的呼叫日志作为输入，构建出一个完整的示例应用。这些日 志至少包含联系过的站点的呼号。呼号是由国家分配的，每个国家都有自己的呼号号段， 所以我们可以根据呼号查到对应的国家。有一些呼叫日志也包含操作者的地理位置，用来帮助确定距离。例 6-1 展示了一段示例日志。本书的示例代码仓库中包含一个需要从呼叫日志中查询并进行处理的呼号列表。

**例 6-1:一条 JSON 格式的呼叫日志示例，其中某些字段已省略**

```json
     {"address":"address here", "band":"40m","callsign":"KK6JLK","city":"SUNNYVALE",
     "contactlat":"37.384733","contactlong":"-122.032164",
     "county":"Santa Clara","dxcc":"291","fullname":"MATTHEW McPherrin",
     "id":57779,"mode":"FM","mylat":"37.751952821","mylong":"-122.4208688735",...}
```

## 6.2 累加器

通常在向Spark传递函数时，可以使用驱动函数定义的变量，但是集群中运行的每个任务都会得到一个变量的副本，更新副本变量不会影响驱动程序中的原值。Spark累加器和广播变量，分别为结果聚合和广播这两种常见通信模式突破了这一限制。

累加器提供了将工作节点的值聚合到驱动程序总的简单语法。累加器的一个常见用途是在调试时对作业执行过程中的事件进行计数。例如，假设我们在 从文件中读取呼号列表对应的日志，同时也想知道输入文件中有多少空行(也许不希望在 有效输入中看到很多这样的行)。例 6-2 至例 6-4 展示了这一场景。

**例 6-2:在 Python 中累加空行**

```python
file = sc.textFile(inputFile)
# 创建Accumulator[Int]并初始化为0 
blankLines = sc.accumulator(0)
def extractCallSigns(line):
		global blankLines # 访问全局变量 
    if (line == ""):
             blankLines += 1
    return line.split(" ")
callSigns = file.flatMap(extractCallSigns)
callSigns.saveAsTextFile(outputDir + "/callsigns")
print "Blank lines: %d" % blankLines.value
```

**例 6-3:在 Scala 中累加空行**

```scala
val sc = new SparkContext(...)
val file = sc.textFile("file.txt")
val blankLines = sc.accumulator(0) // 创建Accumulator[Int]并初始化为0 
val callSigns = file.flatMap(line => {
  if (line == "") {
			blankLines += 1 // 累加器加1
  }
       line.split(" ")
	})
callSigns.saveAsTextFile("output.txt")
println("Blank lines: " + blankLines.value)
```

**例 6-4:在 Java 中累加空行**

```java
JavaRDD<String> rdd = sc.textFile(args[1]);
final Accumulator<Integer> blankLines = sc.accumulator(0);
JavaRDD<String> callSigns = rdd.flatMap(new FlatMapFunction<String, String>() { 
  public Iterable<String> call(String line) { if (line.equals("")) {
             blankLines.add(1);
           }
           return Arrays.asList(line.split(" "));
       }});
callSigns.saveAsTextFile("output.txt")
System.out.println("Blank lines: "+ blankLines.value());
```

累加器的用法总结如下：

- 通 过 在 驱 动 器 中 调 用 SparkContext.accumulator(initialValue) 方 法， 创 建 出 存 有 初 始值的累加器。返回值为 org.apache.spark.Accumulator[T] 对象，其中 T 是初始值 initialValue 的类型。
- Spark闭包里面执行代码可以使用累加器的+=防范(Java中使用add方法)。
- 驱动器程序可以调用累加器的 value 属性(在 Java 中使用 value() 或 setValue())来访问累加器的值。

**注意：**工作节点上的任务不能访问累加器的值，从Job角度看，累加器是一个只写变量。这样的模式下，累加器实现更高效，不需要为每次更新操作进行复杂通信(与Java多线程贡献变量区别)。因为累加器只有驱动程序可以访问，索引累加器校验或其他业务逻辑可以在驱动程序中完成。

现在可以验证呼号，并且只有在大部分输入有效时才输出，使用正则表达式验证。

```python
# 创建用来验证呼号的累加器 
validSignCount = sc.accumulator(0) 
invalidSignCount = sc.accumulator(0)
def validateSign(sign):
  global validSignCount, invalidSignCount
  if re.match(r"\A\d?[a-zA-Z]{1,2}\d{1,4}[a-zA-Z]{1,3}\Z", sign):
    validSignCount += 1
    return True
  else:
    invalidSignCount += 1
    return False

 # 对与每个呼号的联系次数进行计数
validSigns = callSigns.filter(validateSign)
contactCount = validSigns.map(lambda sign: (sign, 1)).reduceByKey(lambda (x, y): x + y) 

# 强制求值计算计数
contactCount.count()
if invalidSignCount.value < 0.1 * validSignCount.value:
         contactCount.saveAsTextFile(outputDir + "/contactCount")
     else:
         print "Too many errors: %d in %d" % (invalidSignCount.value, validSignCount.
     value)
```

### 6.2.1 累加器与容错性

Spark集群对某个工作节点执行失败或执行速度较慢的任务会采取两种方式处理：

- 自动重新执行失败或较慢的任务来应对故障或运行较慢的机器。例如：对某分区执行 map() 操作的节点失败了或者处理速度比别的节点慢，Spark 会在另一个节点上重新运行该任务。
- Spark 推测执行(speculative)机制，在**不同**的executor上再启动一个task来跑这个任务，然后看哪个task先完成，就取该task的结果，并kill掉另一个task。

上面两种场景都会导致同一个函数对同一个数据运行多次，Spark如何保证只会把每个任务对各个累加器的修改应用一次？

**解决方式：**在行动操作中使用的累加器。Spark对于在 RDD 转化操作中使用的累加器，就不能保证只修改一次。举个例子，当一个被缓存下来但是没有经常使用的 RDD 在第一 次从 LRU 缓存中被移除并又被重新用到时，这种非预期的多次更新就会发生。这会强制 RDD 根据其谱系进行重算，而副作用就是这也会使得谱系中的转化操作里的累加器进行更 新，并再次发送到驱动器中。**在转化操作中，累加器通常只用于调试目的**。

### 6.2.2 自定义累加器

前面章节加法操作 Spark 整型累加器类型(Accumulator[Int])。Spark 还直接支持 Double、Long 和 Float 型的累加器。除此以外，Spark 也引入了自定义累加器和聚合操作的 API(比如找到要累加的值中的最大值，而不是把这些值加起来)。自定义累加器需要扩展 **AccumulatorParam**。只要该操作同时满足交换律和结合律，就 可以使用任意操作来代替数值上的加法。

如果对于任意的*a*和*b*，有*a op b* = *b op a*，就说明操作*op*满足交换律。 如 果对于任意的 *a*、*b* 和 *c*，有 (*a op b*) *op c* = *a op* (*b op c*)，就说明操作 *op* 满足 结合律。 例如，sum 和 max 既满足交换律又满足结合律，是 Spark 累加器中的常用操作。

## 6.3 广播变量

广播变量主要用于高效地向所有工作节点发送一个较大的只读值，以供一个或多个 Spark 操作使用。比如，如果你的应用需要向所有节点发送一个较大的只读查询表，甚至是机器学习算法中的一个很大的特征向量。

在此说下Spark闭包变量，Spark会自动把闭包中所有引用到的变量发送到各个工作节点上，与广播变量相比缺点如下：

- 默认的任务发射机制是专门为小任务进行优化的；
- 可能会在多个并行操作中使用同一个变量，但是 Spark 会为每个操作分别发送；
- 如果闭包变量较大，主节点为每个任务分发网络IO浪费严重，且效率低下；

对于需要经常使用的全局变量，比如：经常需要查询的数据，广播变量非常合适，这个值只会被发送到各节点一次，使用的是一种高效的类似 BitTorrent 的通信机制。

例6-6至6-9通使用过呼号的前缀来查询对应的国家进行了对比说明。

**例 6-6:在 Python中使用闭包变量查询国家**

```python
# 查询RDD contactCounts中的呼号的对应位置。将呼号前缀 # 读取为国家代码来进行查询
signPrefixes = loadCallSignTable()
def processSignCount(sign_count, signPrefixes):
  country = lookupCountry(sign_count[0], signPrefixes)
  count = sign_count[1]
  return (country, count)
countryContactCounts = (contactCounts
                        .map(processSignCount)
                        .reduceByKey((lambda x, y: x+ y)))
```

**例 6-7:在 Python 中使用广播变量查询国家**

```python
 # 查询RDD contactCounts中的呼号的对应位置。将呼号前缀 # 读取为国家代码来进行查询
signPrefixes = sc.broadcast(loadCallSignTable())
def processSignCount(sign_count, signPrefixes):
  country = lookupCountry(sign_count[0], signPrefixes.value)
  count = sign_count[1]
  return (country, count)
countryContactCounts = (contactCounts
                        .map(processSignCount)
                        .reduceByKey((lambda x, y: x+ y)))
countryContactCounts.saveAsTextFile(outputDir + "/countries.txt")
```

**例 6-8:在 Scala 中使用广播变量查询国家**

```scala
// 查询RDD contactCounts中的呼号的对应位置。将呼号前缀
// 读取为国家代码来进行查询
val signPrefixes = sc.broadcast(loadCallSignTable())
val countryContactCounts = contactCounts.map{case (sign, count) =>
  val country = lookupInArray(sign, signPrefixes.value)
  (country, count)
}.reduceByKey((x, y) => x + y)
countryContactCounts.saveAsTextFile(outputDir + "/countries.txt")
```

例 6-9:在 Java 中使用广播变量查询国家

```java
// 读入呼号表
 // 查询RDD contactCounts中的呼号对应的国家
final Broadcast<String[]> signPrefixes = sc.broadcast(loadCallSignTable()); JavaPairRDD<String, Integer> countryContactCounts = contactCounts.mapToPair(       
		new PairFunction<Tuple2<String, Integer>, String, Integer> (){
         public Tuple2<String, Integer> call(Tuple2<String, Integer> callSignCount) {
           String sign = callSignCount._1();
           String country = lookupCountry(sign, callSignInfo.value());
           return new Tuple2(country, callSignCount._2());
         }}).reduceByKey(new SumInts());
countryContactCounts.saveAsTextFile(outputDir + "/countries.txt");
```

广播变量的使用过程如下:

- 通过对一个类型 T 的对象调用 SparkContext.broadcast 创建出一个 Broadcast[T] 对象。 任何可序列化的类型都可以这么实现。
- 通过 value 属性访问该对象的值(在 Java 中为 value() 方法)。
- 变量只会被发到各个节点一次，应作为只读值处理(修改这个值不会影响到别的节点)。

注意：广播变量是只读的，尽量在工作节点中不要修改，而在驱动器程序中修改。

### 广播优化

当广播一个比较大的值时，选择既快又好的序列化格式是很重要的，因为如果序列化对象 的时间很长或者传送花费的时间太久，这段时间很容易就成为性能瓶颈。尤其是，Spark 的 Scala 和 Java API 中默认使用的序列化库为 Java 序列化库，因此它对于除基本类型的数组以外的任何对象都比较低效。

解决方式：

-  spark.serializer 属性选择另一个序列化库来 优化序列化过程(第八章会详细介绍)
- 根据类型实现自己的序列化方式(对 Java 对象使用 java.io.Externalizable 接口实现序列化，或使用 reduce() 方法为 Python 的 pickle 库定义自定义的序列化)。

## 6.4 基于分区进行操作

基于分区对数据进行操作可以避免为每个数据元素进行重复的配置工作。诸如打开数据库连接或创建随机数生成器等操作，都是我们应当尽量避免为每个元素都配置一次的工作。Spark提供基于分区的map和foreach方法，让部分代码只对 RDD 的每个分区运行 一次。

以呼号示例程序，有一个在线的业余电台呼号数据库，可以用这个数据库查 询日志中记录过的联系人呼号列表。通过使用基于分区的操作，可以在每个分区内共享一个数据库连接池，来避免建立太多连接，同时还可以重用 JSON 解析器。如例 6-10 至例 6-12 所示，使用 mapPartitions 函数获得输入 RDD 的每个分区中的元素迭代器，而需要返回的是执行结果的序列的迭代器。

**例 6-10:在 Python 中使用共享连接池**

```python
def processCallSigns(signs):
  """使用连接池查询呼号"""
  # 创建一个连接池
  http = urllib3.PoolManager()
  # 与每条呼号记录相关联的URL
  urls = map(lambda x: "http://73s.com/qsos/%s.json" % x, signs) # 创建请求(非阻塞)
  requests = map(lambda x: (x, http.request('GET', x)), urls)
  # 获取结果
  result = map(lambda x: (x[0], json.loads(x[1].data)), requests) # 删除空的结果并返回
  return filter(lambda x: x[1] is not None, result)
  
def fetchCallSigns(input): 
  """获取呼号"""
  return input.mapPartitions(lambda callSigns : processCallSigns(callSigns))

contactsContactList = fetchCallSigns(validSigns)
```

**例 6-11:在 Scala 中使用共享连接池与 JSON 解析器**

``` scala
val contactsContactLists = validSigns.distinct().mapPartitions{
  signs =>
  val mapper = createMapper() 
  val client = new HttpClient() 
  client.start()
  // 创建http请求
  signs.map {sign =>
    createExchangeForSign(sign) 
    // 获取响应
  }.map{ case (sign, exchange) =>
    (sign, readExchangeCallLog(mapper, exchange)) 
  }.filter(x => x._2 != null) // 删除空的呼叫日志
}
```

**例 6-12:在 Java 中使用共享连接池与 JSON 解析器**

```java
// 使用mapPartitions重用配置工作
JavaPairRDD<String, CallLog[]> contactsContactLists =
       validCallSigns.mapPartitionsToPair(
       new PairFlatMapFunction<Iterator<String>, String, CallLog[]>() {
         public Iterable<Tuple2<String, CallLog[]>> call(Iterator<String> input) { 
           //列出结果
           ArrayList<Tuple2<String, CallLog[]>> callsignLogs = new ArrayList<>();
           ArrayList<Tuple2<String, ContentExchange>> requests = new ArrayList<>();
           ObjectMapper mapper = createMapper();
           HttpClient client = new HttpClient();
           try {
             client.start();
             while (input.hasNext()) {
               requests.add(createRequestForSign(input.next(), client));
             }
             for (Tuple2<String, ContentExchange> signExchange : requests) {
               callsignLogs.add(fetchResultFromRequest(mapper, signExchange));
             }
           } catch (Exception e) {
           }
           return callsignLogs;
         }});
System.out.println(StringUtils.join(contactsContactLists.collect(), ","));
```

当基于分区操作RDD时，Spark会为函数提供该分区中元素的迭代器。也会返回一个迭代器。除 mapPartitions() 外，Spark 还有一些别的基于分区的操作符，列在了 表 6-1 中。

​														**表6-1:按分区执行的操作符**

| 函数名                   | 入参                     | 返回值     | 对于RDD[T]的函数签名               |
| ------------------------ | ------------------------ | ---------- | ---------------------------------- |
| mapPartitons( )          | 该分区元素的迭代器       | 元素迭代器 | f: (Iterator[T]) → Iterator[U]     |
| mapPartitonsWithIndex( ) | 分区序号与分区元素迭代器 | 元素迭代器 | f:(Int, Iterator[T] ) →Iterator[U] |
| foreachPartitions()      | 元素迭代器               | 无返回值   | f: (Iterator[T]) → Unit            |

分区操作也可用于避免重复创建对象的开销，时需要创建一个对象来将不同类型的数据聚合起来。回忆一下第 3 章中，当计算平均值时，一种方法是将数值 RDD 转为二元组 RDD，以在归约过程中追踪所处理的元素个数。现在，可以为每个分区只创建一次二元组，而不用为每个元素都执行这个操作，参见例 6-13 和例 6-14。

**例 6-13:在 Python 中不使用 mapPartitions() 求平均值**

```python
def combineCtrs(c1, c2):
    return (c1[0] + c2[0], c1[1] + c2[1])
         
def basicAvg(nums): 
  """计算平均值"""
  nums.map(lambda num: (num, 1)).reduce(combineCtrs)
```

**例 6-14:在 Python 中使用 mapPartitions() 求平均值**

```python
def partitionCtr(nums): 
  """计算分区的sumCounter""" 
  sumCount = [0, 0]
  for num in nums:
    sumCount[0] += num
    sumCount[1] += 1
  return [sumCount]

def fastAvg(nums): 
  """计算平均值"""
  sumCount = nums.mapPartitions(partitionCtr).reduce(combineCtrs)
  return sumCount[0] / float(sumCount[1])
```

## 6.5 外部程序的管道

有三种可用的语言供你选择，这可能已经满足了你用来编写 Spark 应用的几乎所有需求。但 是，如果 Scala、Java 以及 Python 都不能实现你需要的功能，那么 Spark 也为这种情况提供 了一种通用机制，可以将数据通过管道传给用其他语言编写的程序，比如 R 语言脚本。

Spark 在 RDD 上提供 pipe() 方法。Spark 的 pipe() 方法可以让我们使用任意一种语言实现 Spark 作业中的部分逻辑，只要它能读写 Unix 标准流就行。通过 pipe()，你可以 将 RDD 中的各元素从标准输入流中以字符串形式读出，并对这些元素执行任何你需要 的操作，然后把结果以字符串的形式写入标准输出——这个过程就是 RDD 的转化操作过 程。这种接口和编程模型有较大的局限性，但是有时候这恰恰是你想要的，比如在 map 或 filter 操作中使用某些语言原生的函数。

## 6.6 数值RDD操作

Spark 对包含数值数据的 RDD 提供了一些描述性的统计操作。这是我们会在第 11 章介绍的更复杂的统计方法和机器学习方法的一个补充。

Spark的数值操作都是通过流式算法实现的，允许以每次一个元素的方式构建模型，这些 统计数据都会在调用 stats() 时通过一次遍历数据计算出来，并以 StatsCounter 对象返 回。表6-2列出了StatsCounter上的可用方法。

​											**表6-2:StatsCounter中可用的汇总统计数据**

| 方法              | 含义                           |
| ----------------- | ------------------------------ |
| count( )          | RDD中的元素个数                |
| mean( )           | RDD中元素均值                  |
| sum( )            | 总和                           |
| max( )            | 最大值                         |
| min( )            | 最小值                         |
| variance( )       | 方差                           |
| sampleVariance( ) | 采样方差，先采样，再求样本方差 |
| stdev()           | 标准差                         |
| sampleStdev()     | 采样标准差                     |

如果你只想计算这些统计数据中的一个，也可以直接对 RDD 调用对应的方法，比如 rdd. mean() 或者 rdd.sum()。

在例 6-19 至例 6-21 中，我们会使用汇总统计来从数据中移除一些异常值。由于我们会两 次使用同一个 RDD(一次用来计算汇总统计数据，另一次用来移除异常值)，因此应该把 这个 RDD 缓存下来。

**例 6-19:用 Python 移除异常值**

```python
# 要把String类型RDD转为数字数据，这样才能
# 使用统计函数并移除异常值
distanceNumerics = distances.map(lambda string: float(string)) 
stats = distanceNumerics.stats()
stddev = stats.stdev()
mean = stats.mean()
reasonableDistances = distanceNumerics.filter(
       lambda x: math.fabs(x - mean) < 3 * stddev
)
print reasonableDistances.collect()
```

**例 6-20:用 Scala 移除异常值**

```scala
// 现在要移除一些异常值，因为有些地点可能是误报的
// 首先要获取字符串RDD并将它转换为双精度浮点型
val distanceDouble = distance.map(string => string.toDouble)
val stats = distanceDoubles.stats()
val stddev = stats.stdev
val mean = stats.mean
val reasonableDistances = distanceDoubles.filter(x => math.abs(x-mean) < 3 * stddev) println(reasonableDistance.collect().toList)
```

```java
// 首先要把String类型RDD转为DoubleRDD，这样才能使用统计函数
JavaDoubleRDD distanceDoubles = distances.mapToDouble(new DoubleFunction<String>() {
         public double call(String value) {
           return Double.parseDouble(value);
         }});
final StatCounter stats = distanceDoubles.stats();
final Double stddev = stats.stdev();
final Double mean = stats.mean();
JavaDoubleRDD reasonableDistances =distanceDoubles.filter(new Function<Double, Boolean>() {
  public Boolean call(Double x) {
    return (Math.abs(x-mean) < 3 * stddev);}});
System.out.println(StringUtils.join(reasonableDistance.collect(), ","));
```

