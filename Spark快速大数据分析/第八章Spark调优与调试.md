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

Spark 内建的网页用户界面是了解 Spark 应用的行为和性能表现的第一站。默认情况下，在驱动器程序所在机器的 4040 端口上。不过对于YARN集群，应用驱动器程序会运行在集群内部，应该通过 YARN 的资源管理器来访问用户界面。YARN的资源管理器会把请求直接转发给驱动器程序。

Spark用户界面包括几个不同的界面，下面进行详细说明：

**1.作业界面：步骤与任务的进度和指标，以及更多内容**

如下图所示，作业界面包含正在进行的或刚完成不久的Spark作业的详细执行情况。其 中一个很重要的信息是正在运行的作业、步骤以及任务的进度情况。针对每个步骤，这个 页面提供了一些帮助理解物理执行过程的指标。

![](./img/8-2.jpg)

​											**图 8-2:Spark 应用用户界面的作业索引页面**

.本页面经常用来评估一个作业的性能表现。可以着眼于组成作业的所有步骤，看看是不是有一些步骤特别慢，或是在多次运行同一个作业时响应时间差距很大。如果你遇到了 格外慢的步骤，可以点击进去来更好地理解该步骤对应的是哪段用户代码。

确定了需要关注的步骤后，可以借助图8-3的步骤页面定位性能问题，在Spark这样的并行数据处理系统，**数据倾斜**是导致性能问题的常见原因之一。当看到少量 的任务相对于其他任务需要花费大量时间的时候，一般就是发生了数据倾斜。

 ![](./img/8-3.jpg)

​										**图 8-3:Spark 应用用户界面中的步骤详情页面**

**2.存储页面：已经缓存的RDD信息**

当调用了RDD的persist()方法或某个作业中计算了该RDD，这个RDD会被缓存下来。RDD的缓存使用了LRU机制，会移除老RDD。页面展示各个RDD哪些部分被缓存，以及缓存在不同的存储媒介（磁盘、内存等）。

**3.执行器页面：应用中执行器进程列表**

本页面的用处之一在于确认应用可以使用你所预期使用的全部资源量。**调试问题时也最好先浏览这个页面**，因为错误的配置可能会导致启动的执行器进程数量少于预期的，显然也就会影响实际性能。这个页面对于查找行为异常的执行器节点也很有帮助，比如某个执行器节点可能有很高的任务失败率。失败率很高的执行器节点可能表明这 个执行器进程所在的物理主机的配置有问题或者出了故障。只要把这台主机从集群中移 除，就可以提高性能表现。

**4.环境页面：用于调试Spark配置项**

页面枚举了Spark应用运行的环境中实际生效的配置集合。当检查哪些配置标记生效时，这个页面很有用，尤其是同时使用了多种配置机制时。这个页面也会列出添加到应用路径中的所有 JAR 包和文件，在追踪类似**依赖缺失**的问题时可以用到。

### 8.3.2 驱动器进程和执行器进程日志

Spark日志文件的具体位置取决于以下部署方式：

- Spark独立模式：所有日志会在独立模式主节点的网页用户界面直接显示，这些日志默认存储在各个工作节点的Spark目录下的work/目录下。
- 在 Mesos 模式下，日志存储在 Mesos 从节点的 work/ 目录中，可以通过 Mesos 主节点用户界面访问。
- 在 YARN 模式下，最简单的收集日志的方法是使用 YARN 的日志收集工具(运行 yarn logs -applicationId < app ID >)来生成一个包含应用日志的报告。这种方法只有在应用已经完全完成之后才能使用，因为 YARN 必须先把这些日志聚合到一起。要查看 当前运行在 YARN 上的应用的日志，你可以从资源管理器的用户界面点击进入节点(Nodes)页面，然后浏览特定的节点，再从那里找到特定的容器。YARN 会提供对应 容器中 Spark 输出的内容以及相关日志。

Spark的日志系统是基于Log4j实现的，log4j配置的示例文件已经打包在 Spark 中，具体位置是conf/log4j.properties.template。要想自定义 Spark 的日志，首先需要把这个 示例文件复制为log4j.properties，然后就可以修改日志行为了，比如修改根日志等级，默认值为 INFO。以通过 spark-submit 的 --Files 标记添加 log4j.properties 文件。如果在设置日志级别时遇到了困难，请首先确保你没有 在应用中引入任何自身包含 log4j.properties 文件的 JAR 包。Log4j 会扫描整个 classpath， 以其找到的第一个配置文件为准，因此如果在别处先找到该文件，它就会忽略你自定义的 文件。

## 8.4 关键性能考量

本节会进一步讨论Spark运行时可能会遇到的性能方面的常见问题，以及如何调优应用以获取最佳性能。前三小节会介绍如何在代码层面进行改动来提高性能，最后一小节则会讨论如何调优集群 设定以及 Spark 的运行环境。

### 8.4.1 并行度

在物理执行期间，RDD会被分为一系列的分区，每个分区都是整个数据的子集，当Spark调度并运行任务时，Spark会为每个分区的中数据创建出一个任务Task，该任务在默认情况下会需要集群中的 一个计算核心来执行。Spark 也会针对 RDD 直接自动推断出合适的并行度，这对于大多 数用例来说已经足够了。输入 RDD 一般会根据其底层的存储系统选择并行度。例如，从 HDFS 上读数据的输入 RDD 会为数据在 HDFS 上的每个文件区块创建一个分区。从数据混洗后的 RDD 派生下来的 RDD 则会采用与其父 RDD 相同的并行度。

并行度会从两个方面影响程序的性能。首先，当并行度过低时，Spark 集群会出现资源闲置的情况，造成资源浪费。当并行度过高时，每个分区产生 的间接开销累计起来就会更大。评判并行度是否过高的标准包括任务是否是几乎在瞬间(毫秒级)完成的，或者是否观察到任务没有读写任何数据。

Spark提供了两种方法来对操作的并行度调优：

- 数据Shuffle操作时，使用参数的方式为混洗后的RDD指定并行度；
- 对于已有的RDD进行重新分区用来获得更多或者更少的分区，重新分区操作通过 repartition() 实现，该操作会把 RDD 随机打乱并分成设定的分区数目。如果你确定要减少 RDD 分区，可以使用 coalesce() 操作。由于没有打乱数据，该操作比 repartition() 更为高效（coalesce是repartition shuffle=false的实现）。

### 8.4.2 序列化格式

当Spark需要网络传输数据，或将数据溢写到磁盘上时，需要把数据序列化为二进制格式。序列化会在数据进行混洗操作时发生，此时有可能需要通过网络传输大量数据，默认情况下，Spark会使用Java内建的序列化库，Spark 也支持使用第三方序列化库**Kryo**，可以提供比 Java 的序列化工具更短的 序列化时间和更高压缩比的二进制表示，但不能直接序列化全部类型的对象。几乎所有的 应用都在迁移到 Kryo 后获得了更好的性能。

要使用Kryo序列化工具，需要设置spark.serializer 为 org.apache.spark.serializer. KryoSerializer。为了获得最佳性能，你还应该向 Kryo 注册你想要序列化的类，如 例 8-12 所示。注册类可以让 Kryo 避免把每个对象的完整的类名写下来，成千上万 条记录累计节省的空间相当可观。如果你想强制要求这种注册，可以把 spark.kryo. registrationRequired 设置为 true，这样 Kryo 会在遇到未注册的类时抛出错误。

**例 8-12:使用 Kryo 序列化工具并注册所需类**

```scala
val conf = new SparkConf()
conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
// 严格要求注册类
conf.set("spark.kryo.registrationRequired", "true") conf.registerKryoClasses(Array(classOf[MyClass], classOf[MyOtherClass]))
```

不论是选用 Kryo 还是 Java 序列化，如果代码中引用到了一个没有扩展 Java 的 Serializable 接口的类，都会遇到 NotSerializableException。在这种情况下，要查出引发问题的类 是比较困难的，因为用户代码会引用到许许多多不同的类。很多 JVM都支持通过一个特别的选项来帮助调试这一情况:"**-Dsun.io.serialization.extended DebugInfo=true**”。你 可 以 通 过 设 置 spark-submit 的 **--driver-java-options 和 --executor-java-options** 标记来打开这个选项。

### 8.4.3 内存管理

内存对Spark有几种不同的用途：

- **RDD存储**

	当调用 RDD 的 persist() 或 cache() 方法时，这个 RDD 的分区会被存储到缓存区中。 Spark 会根据 spark.storage.memoryFraction 限制用来缓存的内存占整个 JVM 堆空间的 比例大小。如果超出限制，旧的分区数据会被移出内存。

- **数据混序与聚合的缓存区**

	当进行数据混洗操作时，Spakr会创建出一些中间缓存区来存储数据混洗的输出数据，缓存区被用来存储聚合操作的中间结果，以及数据混洗操作中直接输出的部分缓存数据。Spark 会尝试根据 spark.shuffle.memoryFraction 限定这种缓存区内存占总内存的比例。

- **用户代码**

	用户代码的函数可以自行申请大量的内存，例如：应用创建了大数据或对象，用户代码可以访问JVM堆空间中除分配给RDD存储和数据混洗存储以外的全部剩余空间。

在默认情况下，Spark会使用 60%的空间来存储 RDD，20%存储数据混洗操作产生的数据，剩下的 20%留给用户程序。用户可以自行调节这些选项来追求更好的性能表现。

除了调整内存区域比例，还可以改变缓存策略：Spark 默认的 cache() 操作会以 MEMORY_ONLY 的存储等级持久化数据。这意味着如果缓存新RDD分区时，空间不足，旧的分区就会直接被删除。当用到这些分区数据时，再进行重算。 所以有时以 MEMORY_AND_DISK 的存储等级调用 persist() 方法会获得更好的效果，因为在这种存储等级下，内存中放不下的旧分区会被写入磁盘，当再次需要用到的时候再从磁盘上 读取回来。这样的代价有可能比重算各分区要低很多，也可以带来更稳定的性能表现。当 RDD 分区的重算代价很大(比如从数据库中读取数据)时，这种设置尤其有用。

对与默认缓存的策略的另一个改进是缓存序列化后的对象，而非直接缓存。

### 8.4.4 硬件供给

提供给 Spark 的硬件资源会显著影响应用的完成时间。影响集群规模的主要参数包括分配给每个执行器节点的内存大小、每个执行器节点占用CPU核数、执行器节点总数，以及用来存储临时数据的本地磁盘数量。  

**执行器节点内存**

各种部署模式下，执行器节点的内存都可以通过spark.executor.memory配置项或者spark-submit的--executor-memory配置。

**执行器节点CPU核数与执行器节点数量**

YARN模式下，通过spark.executor.cores或spark-submit --executor-cores设置执行器节点CPU核数，通过

--nums-executor设置执行器节点总数。Messos模式和独立部署模式，Spark会从调度器提供的资源中尽可能多的获取CPU核数。过，Mesos 和独立模式也支持通过设置 spark.cores.max 项来限制一个应用中所有执行器节点所使用的核心总数。

**本地磁盘**

用在数据混洗操作中存储临时数据。

**Spark硬件调优的一些建议**

- 集群总内存与CPU核数越多越好

	一般来说，更大的内存和更多的CPU核数对Spark应用会更有用处。Spark框架允许线性伸缩，双倍的资源通常使得应用运行时间减半。在调整集群规模时，还需要将中间计算结果的数据缓存下来，缓存的数据越多，应用的表现越好。

- 本地磁盘越大越好

	Spark需要用到本地磁盘来存储数据混洗操作的中间数据，以及溢写到磁盘的RDD分区数据，因此，使用大量的本地磁盘可以帮助提升 Spark 应用的性能。YARN模式下，由于 YARN 提供了自己的指定临时数据存储目录的机制，Spark的本地磁盘配置项会直接从YARN 的配置中读取。在独立模式和Messos模式下，在 spark-env.sh 文件中设置环境变量 SPARK_LOCAL_DIRS，这样 Spark 应用启动时就会 自动读取这个配置项的值。所有部署模式下，本地目录的设置都应该是单个逗号隔开的目录列表，一般做法是在磁盘的每个分卷中都为Spark设置一个本地目录。写操作会被均衡地分配到所有提供的目录中。**磁盘越多，可以 提供的总吞吐量就越高。**

- 适当的执行器节点内存

	**越多越好**原则在设置执行器节点内存时并不一定适用，使用巨大的堆空间可能会导致GC的长时间暂停，从而严重影响Spark作业的吞肚量。有时，**使用较小内存 (比如不超过 64GB)的执行器实例可以缓解该问题。**Mesos 和 YARN 本身就已经支持在同 一个物理主机上运行多个较小的执行器实例，所以使用较小内存的执行器实例不代表应用所 使用的总资源一定会减少。而在 Spark 的独立模式中，我们需要启动多个工作节点实例(使 用 SPARK_WORKER_INSTANCES 指定)来让单个应用在一台主机上运行于多个执行器节点中。

## 8.5 总结

要深入了解 Spark 调优，请访问官方文档中的调优指南(http://spark.apache.org/docs/latest/tuning.html)。

