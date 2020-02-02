# 第八章 Spark调优与调试

本章介绍如何配置Spark应用，并粗略介绍如何调优和调试生产环境中的Spark工作负载。尽管 Spark 的默认设置在很多情况下无需修改就能直接使用，但有时确实需要修改 某些选项。本章会总结 Spark 的配置机制，着重讨论用户可能需要稍作调整的一些选项。 Spark 的配置对于应用的性能调优是很有意义的。本章的第二部分则涵盖了理解 Spark 应用性能表现的必备基础知识，以及设置相关配置项、编写高性能应用的设计模式等。我们也会讨论 Spark 的用户界面、执行的组成部分、日志机制等相关内容。这些信息在进行性 能调优和问题追踪时都是非常有用的。

## 8.1 使用SparkConf配置Spark

对Spark进行性能调试，通常就是**修改 Spark 应用的运行时配置选项**。Spark 中最主要的配置机制是通过 **SparkConf**类对 Spark进行配置。Spark主要支持三种方式配置SparkConf：

**应用代码中配置**，当创建出一个 SparkContext 时，就需要创建出一个SparkConf的实例，如例 8-1至例 8-3 所示。

**例 8-1:在 Python 中使用 SparkConf 创建一个应用**

```python
conf = SparkConf()
conf.set("spark.app.name", "example8-1")
conf.set("spark.master", "local[1]")
# 重载默认端口配置
conf.set("spark.ui.port", "36000")
# 使用这个配置对象创建一个SparkContext
sc = SparkContext(conf=conf)
```

**例 8-2:在 Scala 中使用 SparkConf 创建一个应用**

```scala
// 创建一个conf对象
val conf = new SparkConf()
conf.set("spark.app.name", "My Spark App") 
conf.set("spark.master", "local[4]") 
conf.set("spark.ui.port", "36000") // 重载默认端口配置
// 使用这个配置对象创建一个SparkContext 
val sc = new SparkContext(conf)
```

**例 8-3:在 Java 中使用 SparkConf 创建一个应用**

```java
// 创建一个conf对象
SparkConf conf = new SparkConf(); 
conf.set("spark.app.name", "My Spark App"); 
conf.set("spark.master", "local[4]"); 
conf.set("spark.ui.port", "36000"); // 重载默认端口配置
// 使用这个配置对象创建一个
SparkContext JavaSparkContext sc = JavaSparkContext(conf);
```

SparkConf 实例包含用户要重载的配置选项的键值对。Spark 中 的每个配置选项都是基于字符串形式的键值对。要使用创建出来的 SparkConf 对象，通过调用 set() 方法来添加配置项的设置，然后把这个对象传给 SparkContext 的构造方法。除了set() 之外，SparkConf 类也包含了一小部分工具方法，可以很方便地设置部分常用参 数。例如，在前面的三个例子中，你也可以调用 setAppName() 和 setMaster() 来分别设置 spark.app.name 和 spark.master 的配置值。

**Spark-submit脚本动态地为应用设置配置选项**。Spark 允许通过 spark-submit 工具动态设置配置项。 当应用被 spark-submit 脚本启动时，脚本会把这些配置项设置到运行环境中。当一个新的SparkConf 被创建出来时，这些环境变量会被检测出来并且自动配好。这样只需要在Spark Application新建空SparkConfig就好。

例 8-4:在运行时使用标记设置配置项的值

```sh
     $ bin/spark-submit \
       --class com.example.MyApp \
       --master local[4] \
       --name "My Spark App" \
       --conf spark.ui.port=36000 \
       myApp.jar
```

**Spark-submit脚本从配置文件中读取配置项。**这对于设置一些与环境相关的配置项比较有用，方便不同用户共享这些配置(比如默认的 Spark 主节点)。默认情况下，spark- submit脚本会在 Spark 安装目录中找到 **conf/spark-defaults.conf 文件**，尝试读取该文件中以空格隔开的键值对数据。你也可以通过spark-submit的 --properties-File 标记，自定义该文件的路径，如例 8-5 所示。

**例 8-5:运行时使用默认文件设置配置项的值**

```sh
$ bin/spark-submit \
	--class com.example.MyApp \
	--properties-file my-config.conf \
	myApp.jar
	
## Contents of my-config.conf ##
spark.master    local[4]
spark.app.name  "My Spark App"
spark.ui.port   36000
```

**Tips**

SparkConf一旦传给了 SparkContext 的构造方法，应用所绑定的 SparkConf 就不可变了。 这意味着所有的配置项都必须在 SparkContext 实例化出来之前定下来。

SparkConf三种配置方式的生效优先级：优先级最高的是在用户代码中显式调用 set() 方法设置的选项；其次是通过 spark-submit 传递的参数；再次是写在配置文件中 的值，最后是系统的默认值。

​				表 7-2 列出了一些常用的配置项。表 8-1 则列出了其他几个比较值得关注的配置项。

| 选项                                                         | 默认值                                       | 描述                                                         |
| ------------------------------------------------------------ | -------------------------------------------- | ------------------------------------------------------------ |
| spark.executor.memory (--executor-memory)                    | 512m                                         | 为每个执行器进程分配的内存，格式与JVM内存字符串格式一样(例如 512m，2g)。关于本配置项的更多细节请参阅 8.4.4 节 |
| spark.executor.cores(--executor-cores) spark.cores.max (--total-executor-cores) | 1                                            | 限制应用使用的核心个数的配置项。在 YARN 模式下， spark.executor.cores 会为每个任务分配指定数目的核心。在独立模式和Mesos模式下，spark.core.max设置所有执行器进程使用的核心总数的上限。 |
| spark.speculation                                            | False                                        | 设为 true 时开启任务预测执行机制。当出现比较慢的任务时，这种机制会在另外的节点上也尝试执行该任务的一个副本。打开此选项会帮助减少大规模集群中个别较慢的任务带来的影响。 |
| spark.storage.blockMan agerTimeoutIntervalMs                 | 45000                                        | 内部用来通过超时机制追踪执行器进程是否存活的阈值。 对于会引发长时间垃圾回收(GC)暂停的作业，需要 把这个值调到 100 秒(对应值为 100000)以上来防止失 败。在 Spark 将来的版本中，这个配置项可能会被一个 统一的超时设置所取代，所以请注意检索最新文档 |
| spark.executor.extraJava Options, spark.executor.extra ClassPath, spark.executor.extra LibraryPath | (空)                                         | 这三个参数用来自定义如何启动执行器进程的 JVM， 分别用来添加额外的 Java 参数、classpath 以及程序库路径。使用字符串来设置这些参数(例如 spark. executor.extraJavaOptions="- XX:+PrintGCDetails- XX:+PrintGCTimeStamps")。请注意，虽然这些参数可以让你自行添加执行器程序的 classpath，但是推荐使用 spark-submit 的 --jars 标记来添加依赖，而不是使用这 几个选项 |
| spark.serializer                                             | org.apache.spark. serializer. JavaSerializer | 指定用来进行序列化的类库，包括通过网络传输数据或 缓存数据时的序列化。默认的 Java 序列化对于任何可 以被序列化的 Java 对象都适用，但是速度很慢。我们 推 荐 在 追 求 速 度 时 使 用 org.apache.spark.serializer. KryoSerializer 并且对 Kryo 进行适当的调优。该项可以 配置为任何 org.apache.spark.Serializer 的子类 |
| spark.[X].port                                               | (任意值)                                     | 用来设置运行 Spark 应用时用到的各个端口。这些参数 对于运行在可靠网络上的集群是很有用的。有效的 X 包 括 driver、fileserver、broadcast、replClassServer、 blockManager，以及 executor |
| spark.eventLog.enabled                                       | false                                        | 设为 true 时，开启事件日志机制，这样已完成的 Spark 作业就可以通过历史服务器(history server)查看。 |
| spark.eventLog.dir                                           | file:///tmp/ spark-events                    | 指开启事件日志机制时，事件日志文件的存储位置。这 个值指向的路径需要设置到一个全局可见的文件系统中， 比如 HDFS |

但有一个重要的选项是个例外。 需要在 conf/spark-env.sh 中将环境变量 SPARK_LOCAL_DIRS 设置为用逗号隔开的存储位置列表，来指定 Spark 用来混洗数据的本地存储路径。这需要在独立模式和 Mesos 模式下设置。8.4.4 节中会更详细地介绍 SPARK_LOCAL_DIRS 的相关知识点。这个配置项之所以和其他 的 Spark 配置项不一样，是因为它的值在不同的物理主机上可能会有区别。

## 8.2 Spark执行的组成部分：作业、任务和步骤

在执行时，Spark会把多个操作合并为一组任务，把RDD的逻辑表示翻译为物理执行计划。下面通过一个示例应用来展示 Spark 执行的各个阶段，以了解用户代码如何被编译为下层的执行计划。我们要使用的是一个使用 Spark shell 实现的简单的日志分析应用。输入数据是一个由不同严重等级的日志消息和一些分散的空行组成的文本文件，如例 8-6 所示。

**例 8-6:用作示例的源文件 input.txt**

```txt
## input.txt ##
INFO This is a message with content
INFO This is some other content
(空行)
INFO Here are more messages
WARN This is a warning
(空行)
ERROR Something bad happened
WARN More details on the bad thing
INFO back to normal messages
```

计算其中各级别的日志消息的条数。

```scala
// 读取输入文件
scala> val input = sc.textFile("input.txt") // 切分为单词并且删掉空行
scala> val tokenized = input.
          |   map(line => line.split(" ")).
| filter(words => words.size > 0)
// 提取出每行的第一个单词(日志等级)并进行计数 
scala> val counts = tokenized.
          |   map(words = > (words(0), 1)).
          | reduceByKey{ (a,b) => a + b }
```

程序定义了一个RDD对象的**有向无环图(DAG)**，可以在稍后行动操作被触发时用它来进行计算。每个 RDD 维护了其指向一个或多个父节点的引用，以及表示其与父节点之间关系的信息。比如，当在 RDD 上调用 val b = a.map() 时，b这个RDD就存下了对其父节点a的一个引用。这些引用使得 RDD 可以追踪到其所有的祖先节点。

Spark 提供了 toDebugString() 方法来查看 RDD 的谱系。在例 8-8 中，会看到在前面例子中创建出的一些 RDD。

**例 8-8:在 Scala 中使用 toDebugString() 查看 RDD**

```scala
     scala> input.toDebugString
     res85: String =
     (2) input.text MappedRDD[292] at textFile at <console>:13
       | input.text HadoopRDD[291] at textFile at <console>:13
     scala> counts.toDebugString
     res84: String =
     (2) ShuffledRDD[296] at reduceByKey at <console>:17
      +-(2) MappedRDD[295] at map at <console>:17
         |  FilteredRDD[294] at filter at <console>:15
         |  MappedRDD[293] at map at <console>:15
         |  input.text MappedRDD[292] at textFile at <console>:13
         |  input.text HadoopRDD[291] at textFile at <console>:13
```

可以看到 sc.textFile() 方法所创建出的 RDD 类型，找到关于 textFile() 幕后细节的一些蛛丝马迹。事实上，该方法创建出了一个 HadoopRDD 对象，然后对该 RDD 执行映射操作，最终得到了返回的 RDD。counts 的谱系图则更加复杂一些，有多个祖先 RDD，这是由于我们对 input 进行了其他操作，包括额外的映射、筛选以及归约。

在调用行动操作之前，RDD 都只是存储着可以让我们计算出具体数据的描述信息。要触发 实际计算，需要对 counts 调用一个行动操作，比如使用 collect() 将数据收集到驱动器程 序中，如例 8-9 所示。

**例 8-9:收集 RDD**

```scala
scala> counts.collect()
				res86: Array[(String, Int)] = Array((ERROR,1), (INFO,4), (WARN,2))
```

Spark Executor调度器会创建出用于计算行动操作的 RDD 物理执行计划。此处调用 RDD的collect() 方法，于是 RDD 的每个分区都会被物化出来并发送到驱动器程序中。Spark 调度器从最终被调用行动操作的 RDD(在本例中是 counts)出发，向上回溯所有必须计算 的 RDD。调度器会访问 RDD 的父节点、父节点的父节点，以此类推，递归向上生成计算所有必要的祖先 RDD的物理计划。

**最简单的情况**：调度器为有向图中的每个RDD输出计算步骤，即一个RDD对应一个步骤；步骤中包括 RDD 上需要应用于每个分区的任务。然后以相反的顺序执行这些步骤，计算得出最终所求的 RDD。

**复杂情况：**此时RDD图与执行步骤的对应关系不一定是一一对应的，比如：当调度器进行流水线执行计划（pipelining），或把多个RDD合并到一个步骤时。当RDD不需要Shuffle数据就可以直接从父节点计算，即当前RDD与父RDD的依赖关系为窄依赖，调度器就会自动进行流水线执行。**例8-8 中输出的谱系图使用不同缩进等级来展示 RDD 是否会在物理步骤中进行流水线执行。**在物理执行时，执行计划输出的缩紧等级与其父节点相同的RDD会与其父节点在同一个步骤中流水线执行。

例如，当计算 counts 时，尽管有很多级父 RDD，但从缩进来看 总共只有两级。这表明物理执行只需要两个步骤。由于执行序列中有几个连续的筛选和 映射操作，这个例子中才出现了流水线执行。图 8-1 的右半部分展示了计算 counts 这个 RDD 时的两个执行步骤。

<img src="./img/8-1.jpg" style="zoom:75%;" />

​									**图 8-1:将 RDD 转化操作串联成物理执行步骤**

当一个RDD已经缓存在集群内存或磁盘上时，Spark的内部调度器也会自动截短RDD谱系图，直接基于缓存的 RDD进行计算，不会上溯到缓存RDD的祖先。还有一种截短 RDD 谱系图的情况发生在当 RDD 已经在之前的数据混洗中作为副产品物化出来时，这种内部优化是基于Spark数据混洗操作输出均被写入磁盘的特性，同时也充分利用了RDD图的某些部分会被多次计算的事实。

下面来看看物理执行中缓存的作用。我们把 countsRDD 缓存下来，观察之后的行动操作是怎样被截短的(例 8-10)。如果再次访问用户界面，会看到缓存减少了后来执行计 算时所需要的步骤。多次调用 collect() 只会产生一个步骤来完成这个行动操作。

**例 8-10:计算一个已经缓存过的 RDD**

```scala
// 缓存RDD
scala> counts.cache()
// 第一次求值运行仍然需要两个步骤
scala> counts.collect()
res87: Array[(String, Int)] = Array((ERROR,1), (INFO,4), (WARN,2), (##,1), ((empty,2))
// 该次执行只有一个步骤
scala> counts.collect()
res88: Array[(String, Int)] = Array((ERROR,1), (INFO,4), (WARN,2), (##,1), ((empty,2))
```

特定的行动操作所生成的步骤的集合被称为一个作业，作业由一个或者多个步骤组成。

一旦步骤图确定下来，任务就会被创建出来并发给内部的调度器，该调度器在不同的部署 模式下会有所不同。物理计划中的步骤会依赖于其他步骤，如RDD谱系图所显示。

一个物流步骤会启动很多任务，每个任务都是在不同的数据分区上做同样的事情，任务内部流程是一样的，如下所述：

- 从数据存储（如果该RDD是一个输入RDD）或已有RDD（基于已经缓存的数据）或数据混洗的输出中获取输入数据；
- 执行必要的操作来计算出这些操作代表的RDD，例如：对输入数据执行 filter() 和 map() 函数，或者进行分组或归约操作。
- 把输出写入到一个数据混洗文件中，写入外部存储，或者发回驱动器程序（如果最终 RDD 调用的是类似 count() 这样的行动操作）。

Spark大部分的日志信息和工具都是以步骤、任务或数据混洗为单位的。总结归纳下Spark应用执行流程：

- **应用代码定义RDD的DAG有向无环图**：RDD 上的操作会创建出新的 RDD，并引用它们的父节点，这样就创建出了一个图。
- **行动操作把有向无环图转译为执行计划**：当调用行动操作时，这个RDD就必须被计算出来，该RDD的父节点也必须被计算出来，Spark调度器提交一个作业来计算所有必要的RDD，这个作业会包含 一个或多个步骤，每个步骤其实也就是一波并行执行的计算任务。一个步骤对应有向无环图中的一个或多个 RDD，一个步骤对应多个 RDD 是因为发生了流水线执行。
- **任务于集群中调度并行：**步骤是按顺序处理的，任务则独立地启动来计算出 RDD 的一部分。一旦作业的最后一 个步骤结束，一个行动操作也就执行完毕了。

## 8.3 查找信息

Spark在应用执行时记录详细的进度信息和性能指标，可以从两个地方找到：park 的网页用户界面以及驱动器进程和执行器进程生成的日志文件。

### 8.3.1 Spark网页用户界面

Spark 内建的网页用户界面是了解 Spark 应用的行为和性能表现的第一站。默认情况下，在驱动器程序所在机器的 4040 端口上。不过对于YARN集群，应用驱动器程序会运行在集群内部，应该通过 YARN 的资源管理器来访问用户界面。YARN 的资源 管理器会把请求直接转发给驱动器程序。

