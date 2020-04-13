# 第七章 在集群上运行Spark

## 7.1 简介

Spark的一大好处就是可以通过增加机器数量并使用集群模式运行，来扩展程序的计算能力。编写用于在集群上并行执行的 Spark 应用所使用的 API 跟本书之前 几章所讨论的完全一样。也就是说，你可以在小数据集上利用本地模式快速开发并验证你 的应用，然后无需修改代码就可以在大规模集群上运行。

本章首先介绍分布式 Spark 应用的运行环境架构，然后讨论在集群上运行 Spark 应用时的 一些配置项。Spark 可以在各种各样的集群管理器(Hadoop YARN、Apache Mesos，还有 Spark 自带的独立集群管理器)上运行，所以 Spark 应用既能够适应专用集群，又能用于 共享的云计算环境。我们会对各种使用情况下的优缺点和配置方法进行探讨。同时，也会 讨论 Spark 应用在调度、部署、配置等各方面的一些细节。读完本章之后，你应该能学会 运行分布式 Spark 程序的一切技能。

## 7.2 Spark运行时架构

在分布式环境下，Spark集群采用的是主/从架构，一个节点负责中央协调，调度各个分布式工作节点。这个中央协调节点被称为驱动器(Driver)节点，工作节点成为执行器(Executor)节点。驱动器节点可以和大量的执行器节点进行通信，它们都作为独立的 Java 进程运行。驱动器节点和所有的执行器节点一起被称为一个 Spark 应用(application)。

![](./img/7-1.jpg)

​				 											图 7-1:分布式 Spark 应用中的组件

Spark应用通过一个叫集群管理器(Cluster Manager)的外部服务在集群中的机器上启动，Spark 自带的集群管理器被称为独立集群管理器(Standalone模式)。Spark 也能运行在 Hadoop YARN 和 Apache Mesos 这两大开源集群管理器上。

### 7.2.1 驱动器节点

Spark驱动器（Driver）是执行程序中的main方法进程，执行用户创建SparkContext、创建RDD、RDD转换操作和行动操作的代码。Spark shell模式下，就启动了一个 Spark驱动器程序(Spark shell 总是 会预先加载一个叫作 sc 的 SparkContext 对象)。驱动器程序一旦终止，Spark应用也就结束了。

Spark驱动器程序(Driver)有两个主要职责：

- **把Driver程序转为任务**

	Spark驱动器程序负责将用户程序转换为多个物理执行单元，执行单元被称为任务（Task）。所有的Spark Application都遵循同样的架构：程序从输入数据创建一系列RDD，使用转换操作派生出新RDD，行动操作收集或存储结果RDD中的数据。Spark Application隐式地创建出了一个由操作组成的有向无环图（Directed Acyclic Graph，简称 DAG），当程序执行时，将DAG转换为物理执行计划。

	 Spark会对逻辑执行计划进行一系列优化，比如：将连续映射转换为**流水线**化执行，将多个操作合并到一个步骤中等。Spark逻辑计划转换为一系列步骤（Stage），每个步骤由多个任务组成，这些任务会被打包并送到集群中。任务是 Spark 中最小的工作单 元，用户程序通常要启动成百上千的独立任务。

- **为执行器(Executor)节点调度任务**

	有了物理执行计划，Spark Driver程序必须在各个Executor执行器进程中协调Task任务调度。执行器启动后，会向驱动器进程注册自己，因此，驱动器进程始终对应用中所有的执行 器节点有完整的记录。每个执行器节点代表一个能够处理任务和存储 RDD 数据的进程。

	Spark驱动器程序会根据当前的执行器节点集合，尝试把所有任务基于数据所在位置分配合适的Executor执行器进程。当任务执行时，执行器进程会把缓存数据存储起来，而驱动器 进程同样会跟踪这些缓存数据的位置，并且利用这些位置信息来调度以后的任务，以尽 量减少数据的网络传输。

### 7.2.2 执行器节点

Spark Executor执行器节点是一个工作进程，负责在Spark作业中运行任务，任务之间相互独立。Spark 应用启动时，执行器节点就被同时启动，并且始终伴随着整个Spark应用的生命周 期而存在。如果有执行器节点发生了异常或崩溃，Spark 应用也可以继续执行。执行器进 程有两大作用：

- 负责运行组成Spark Application的Task任务，并将结果返回给驱动器进程；
- 通过自身的块管理器（Block Manager）为用户程序中要求缓存的RDD提供内存式的存储，RDD是直接缓存在Executor执行器进程内的，因此任务可以在运行时充分利用缓存数据加速/迭代运算。

**Tips 本地模式中的驱动器程序和执行器程序**

在本地模式下，Spark 驱 动器程序和各执行器程序在同一个 Java 进程中运行。本地模式主要用于程序开发与调试，从线上数据集中抽取部分数据，在本地开发验证Spark Driver程序。

### 7.2.3 集群管理器

Spark依赖于集群管理器来启动执行器节点，除了 Spark 自带的独立集群管理器，Spark 也可以运行在其他外部集群管理器上，比 如 YARN 和 Mesos。

Spark 文档中始终使用**驱动器节点**和**执行器节点**的概念来描述执行 Spark Application的进程。而**主节点(master)**和**工作节点(worker)**的概念则被用来分别表述**集群管理器**中的**中心化的部分和分布式的部分**。这些概念很容易混淆，所以要格外小心。例如，Hadoop YARN 会启动一个叫作资源管理器(Resource Manager)的主节点守护进程，以及一系列叫作节点管理器(Node Manager)的工作节点守护进程。而在 YARN 的工作节点上，Spark 不仅可 以运行执行器进程，还可以运行驱动器进程。

### 7.2.4 启动一个程序

不论使用哪一种集群管理器，都可以使用 Spark 提供的统一脚本 spark-submit 将应用提交到集群管理器上。通过不同的配置选项，spark-submit 可以连接到相应的集群管理器上，并控制应用所使用的资源数量。在使用某些特定集群管理器时，spark- submit 也可以将驱动器节点运行在集群内部(比如一个 YARN 的工作节点)。但对于其他的集群管理器，驱动器节点只能被运行在本地机器上。我们会在下一节更加详细地讲解 spark-submit。

### 7.2.5 总结

- 用户通过spark-submit脚本提交应用；
- spark-submit脚本启动Driver驱动器程序，通过调用Java、Python和Scala的main方法；
- Driver驱动器程序与Cluser Manager集群管理器通信，申请资源以启动执行器节点；
- Cluser Manager集群管理器为Driver驱动器程序启动Executor执行器节点；
- Driver驱动器进程执行用户Application的操作，根据程序中所定义的对RDD的转换操作和行动操作，驱动器节点把工作以任务的方式发送到执行器进程中；
- 任务在执行器中进行计算并保存结果
-   如果驱动器程序的 main() 方法退出，或者调用了 SparkContext.stop()，驱动器程序会终止执行器进程，并且通过集群管理器释放资源。

## 7.3 使用spark-submit部署应用

如果在调用 spark-submit 时除了脚本或 JAR 包的名字之外没有别的参数，那么这个 Spark 程序只会在本地执行。

**例 7-1:提交 Python 应用**

```shell
bin/spark-submit my_script.py
```

如果要将应用提交到 Spark 独立集群上，可以将独立集群的地址和希望启动的每个执行器进程的大小作为附加标记提供，如例 7-2 所示。

**例 7-2:提交应用时添加附加参数**

```shell
bin/spark-submit 
		--master spark://host:7077 \\
		--executor-memory 10g \\
    my_script.py
```

--master标记指定要链接的Spark集群，可以支持Spark://、local、mesos等多种类型集群，如表7-1所示：

​									**表7-1:spark-submit的--master标记可以接收的值**

| 值                | 描述                                                         |
| ----------------- | ------------------------------------------------------------ |
| spark://host:port | 连接到指定端口的 Spark 独立集群上。默认情况下 Spark 独立主节点使用 7077 端口 |
| Mesos://host:port | 连接到指定端口的 Mesos 集群上。默认情况下 Mesos 主节点监听 5050 端口 |
| yarn              | 连接到一个 YARN 集群。当在 YARN 上运行时，需要启动Hadoop的YARN服务并设置环境变量 HADOOP _CONF_DIR 指向 Hadoop 配置目录，以获取集群信息 |
| local             | 运行本地模式，使用单核                                       |
| local[N]          | 运行本地模式，使用 N 个核心                                  |
| local[*]          | 运行本地模式，使用尽可能多的核心（取决于Spark被分配的CPU核数） |

spark-submit提供了各种选项，可以控制应用每次运行的各项细节。这些选项主要分为两类：

- 第一类是调度信息，比如你希望为作业申请的资源量(如例 7-2 所示)。
- 第二类是应用的运行时依赖，比如需要部署到所有工作节点上的库和文件。

spark-submit的一般格式见例 7-3。 

**例 7-3:spark-submit 的一般格式**

```shell
bin/spark-submit [options] <app jar | python file> [app options]
```

里面主要分为三部分：

- [options] 是要传给 spark-submit 的标记列表。你可以运行 spark-submit -- help列出可以接收的标记。表 7-2 列出了一些常见的标记。

	​														**表7-2:spark-submit的一些常见标记**

	| 标记              | 描述                                                         |
	| ----------------- | ------------------------------------------------------------ |
	| --master          | 表示要连接的集群管理器。这个标记可接收的选项见表 7-1         |
	| --deploy-mode     | client表示本地客户端启动Driver，cluster表示在集群中一台工作节点机器上启动。client模式下，spark-submit将Driver运行被调用的机器上。cluster模式下，驱动器程序会被传输并执行于集群的一个工作节点上。默认是本地模式 |
	| --class           | 运行 Java 或 Scala 程序时应用的主类                          |
	| --name            | 应用的显示名，会显示在 Spark 的网页用户界面中                |
	| --jars            | 需要上传并放到应用的 CLASSPATH 中的 JAR 包的列表。如果应用依赖于少量第三方的 JAR 包，可以把它们放在这个参数里 |
	| --files           | 需要放到应用工作目录中的文件的列表。这个参数一般用来放需要分发到各节点的 数据文件 |
	| --py-files        | 需要添加到 PYTHONPATH 中的文件的列表。其中可以包含 .py、.egg 以及 .zip 文件 |
	| --executor-memory | 执行器进程使用的内存量，以字节为单位。可以使用后缀指定更大的单位，比如 “512m”(512 MB)或“15g”(15 GB) |
	| --driver-memory   | 驱动器进程使用的内存量，以字节为单位。可以使用后缀指定更大的单位，比如 “512m”(512 MB)或“15g”(15 GB) |

- <app jar | python File> 表示包含应用入口的 JAR 包或 Python 脚本

- [app options] 是传给你的应用的选项。如果你的程序要处理传给 main() 方法的参数（args），它 只会得到 [app options] 对应的标记，不会得到 spark-submit 的标记。

spark-submit还允许通过--conf prop=value标记设置任意SparkConf配置，也可以使用--properties-File指定一个包含键值对的属性文件。第 8 章会讨论 Spark 的配置系统。

例 7-4 展示了一些使用各种选项调用 spark-submit 的例子，这些调用语句都比较长。 例 7-4:使用各种选项调用 spark-submit

```
# 使用独立集群模式提交Java应用 
$ ./bin/spark-submit \  
    --master spark://hostname:7077 \
    --deploy-mode cluster \
    --class com.databricks.examples.SparkExample \
    --name "Example Program" \
    --jars dep1.jar,dep2.jar,dep3.jar \
    --total-executor-cores 300 \
    --executor-memory 10g \
    myApp.jar "options" "to your application" "go here"
       
# 使用YARN客户端模式提交Python应用
$ export HADOP_CONF_DIR=/opt/hadoop/conf 
$ ./bin/spark-submit \
	--master yarn \
	--py-files somelib-1.2.egg,otherlib-4.4.zip,other-file.py \
	--deploy-mode client \
	--name "Example Program" \
  --queue exampleQueue \
  --num-executors 40 \
  --executor-memory 10g \
  my_script.py "options" "to your application" "go here"
```

## 7.4 打包代码与依赖

通常用户程序则需要依赖第三方的库。如果程序引入了任何既不在 org.apache.spark 包内也不属于语言运行 时的库的依赖，你就需要确保所有的依赖在该 Spark 应用运行时都能被找到。

- Python

	Python多种安装第三方库的方法。由于PySpark使用工作节点机器上已有的 Python 环境，你可以通过标准的 Python 包管理器(比如 pip 和 easy_install)直接在集群中的所有机器上安装所依赖的库，或者把依赖手动安装到 Python 安装目录下的 site- packages/ 目录中。也可以使用 spark-submit 的 --py-Files 参数提交独立的库，将其添加到 Python 解释器的路径中。如果没有在集群上安装包的权限，可以手动添加依赖库，但是要防范与已经安装在集群上的那些包发生冲突。

- Java与Scala

	Java 和 Scala 也可以通过 spark-submit 的 --jars 标记提交独立的 JAR 包依赖。当只有一两个库的简单依赖，并且这些库本身不依赖于其他库时，这种方法比较合适。但是一般 Java 和 Scala 的工程会依赖很多库。当向 Spark 提交应用时，必须把应用的所有依赖都传给集群。不仅要传递你直接依赖的库，还要传递这些库的依赖，以及它们的依赖的依赖，等等。手动维护和提交全部的依赖 JAR 包是很笨的方法。事 实上，常规的做法是使用构建工具，例如：Maven，sbt，Maven 通常用于 Java 工程，而 sbt 则一般用于 Scala 工程。生成单个大 JAR 包，包含应用的所有的传递依赖。这 通常被称为超级(uber)JAR 或者组合(assembly)JAR。

### 7.4.1 使用Maven构建用Java编写的Spark程序

下面来看一个有多个依赖的 Java 工程，在示例中我们会将它打包为一个超级 JAR 包。例 7-5 提供了 Maven 的 pom.xml 文件。

​			**例 7-5:使用 Maven 构建的 Spark 应用的 pom.xml 文件**

```xml
<project>
       <modelVersion>4.0.0</modelVersion>
  		 <!-- 工程相关信息 --> 
  		 <groupId>com.databricks</groupId> 
  		 <artifactId>example-build</artifactId> 
  		 <name>Simple Project</name> 
       <packaging>jar</packaging> 
       <version>1.0</version>
			<dependencies>
        <!-- Spark依赖 --> 
        <dependency>
           <groupId>org.apache.spark</groupId>
           <artifactId>spark-core_2.10</artifactId>
           <version>1.2.0</version>
           <!-- 标记为 provided确保Spark不与应用依赖的其他工件打包在一起-->
           <scope>provided</scope>
        </dependency> 
        <!-- 第三方库 --> 
        <dependency>
           <groupId>net.sf.jopt-simple</groupId>
           <artifactId>jopt-simple</artifactId>
           <version>4.3</version>
        </dependency> 
        <!-- 第三方库 -->
        <dependency>
           <groupId>joda-time</groupId>
           <artifactId>joda-time</artifactId>
           <version>2.0</version>
         </dependency>
       </dependencies>
       <build>
         <plugins>
           <!-- 用来创建超级JAR包的Maven shade插件 --> 
           <plugin>
             <groupId>org.apache.maven.plugins</groupId>
             <artifactId>maven-shade-plugin</artifactId>
             <version>2.3</version>
             <executions>
               <execution>
                 <phase>package</phase>
                 <goals>
                   <goal>shade</goal>
                   </goals>
               </execution>
             </executions>
           </plugin>
         </plugins>
       </build>
</project>
```

### 7.4.2 使用sbt构建的用Scala编写的Spark应用

sbt 是一个通常在 Scala 工程中使用的比较新的构建工具。sbt的目录结构和 Maven 相 似。在工程的根目录中，要创建出一个叫作 build.sbt 的构建文件，源代码则应该放在 src/main/scala 中。sbt构建文件是用配置语言写成的，在这个文件中把值赋给特定的键，用来定义工程的构建。例如，有一个键叫作 name，是用来指定工程名字的，还有一个 键叫作 libraryDependencies，用来指定工程的依赖列表。

**例 7-7:使用 sbt 0.13 的 Spark 应用的 build.sbt 文件**

```sbt
import AssemblyKeys._
name := "Simple Project"
version := "1.0"
organization := "com.databricks"
scalaVersion := "2.10.3"
libraryDependencies ++= Seq( // Spark依赖
	"org.apache.spark" % "spark-core_2.10" % "1.2.0" % "provided", // 第三方库
	"net.sf.jopt-simple" % "jopt-simple" % "4.3",
	"joda-time" % "joda-time" % "2.0"
)
// 这条语句打开了assembly插件的功能 
assemblySettings
// 配置assembly插件所使用的JAR
jarName in assembly := "my-project-assembly.jar"
// 一个用来把Scala本身排除在组合JAR包之外的特殊选项，因为Spark // 已经包含了Scala
assemblyOption in assembly :=
       (assemblyOption in assembly).value.copy(includeScala = false)

```

这个构建文件的第一行从插件中引入了一些功能，这个插件用来支持创建项目的组合 JAR 包。要使用这个插件，需要在 project/ 目录下加入一个小文件，来列出对插件的依赖。我 们只需要创建出 project/assembly.sbt 文件，并在其中加入 addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "0.11.2")。sbt-assembly 的实际版本可能会因使用的 sbt 版本不同而变 化。例 7-8 适用于 sbt 0.13。

例 7-8:在 sbt 工程构建中添加 assembly 插件

```shell
# 显示project/assembly.sbt的内容
$ cat project/assembly.sbt
addSbtPlugin("com.eed3si9n" % "sbt-assembly" % "0.11.2")
```

定义好了构建文件之后，就可以创建出一个完全组合打包的 Spark 应用 JAR 包。

### 7.4.3 依赖冲突

当用户应用与Spark本身依赖同一个库时可能会发生依赖冲突，导致程序崩溃。依赖冲突通常表现为Spark 作业执行过程中抛出 NoSuchMethodError、ClassNotFoundException，或其他与类加载相关的JVM异常。主要有两种解决方式：

- 修改应用与Spark所依赖的库版本名称相同；

- 使用通常被称为**“shading”**的方式打包你的应用。Maven构建工具通过对例 7-5 中使用的插件(shading 的功能也是这个插件取名为maven- shade-plugin的原因)进行高级配置来支持这种打包方式。shading以另一个命名空间保留冲突的包，并自动重写应用的代码使得它们使用重命名后的版本。这种技术有些 简单粗暴，不过对于解决运行时依赖冲突的问题非常有效。

	**下面的配置将org.codehaus.plexus.util jar 包重命名为org.shaded.plexus.util。**

	```xml
	<build>
	    <plugins>
	      <plugin>
	        <groupId>org.apache.maven.plugins</groupId>
	        <artifactId>maven-shade-plugin</artifactId>
	        <version>2.4.3</version>
	        <executions>
	          <execution>
	            <phase>package</phase>
	            <goals>
	              <goal>shade</goal>
	            </goals>
	            <configuration>
	              <relocations>
	                <relocation>
	                  <pattern>org.codehaus.plexus.util</pattern>
	                  <shadedPattern>org.shaded.plexus.util</shadedPattern>
	                  <excludes>
	                    <exclude>org.codehaus.plexus.util.xml.Xpp3Dom</exclude>
	                    <exclude>org.codehaus.plexus.util.xml.pull.*</exclude>
	                  </excludes>
	                </relocation>
	              </relocations>
	            </configuration>
	          </execution>
	        </executions>
	      </plugin>
	    </plugins>
	  </build>
	```

## 7.5 Spark应用内和应用间的调度

现实中，许多集群是多个用户之间共享的，共享的环境会带来调度方面的挑战:如果两个用户都启动了希望使用整个集群所有资 源的应用，该如何处理呢? 

Spark 有一系列调度策略来保障资源不会被过度使用，还允许工作负载设置优先级。在调度多个用户共享的集群时，Spark主要依赖集群管理器来在Spark应用间共享资源。应用收到的执行器节点个数可能比它申请的更多或者 更少，这取决于集群的可用性与争用。许多集群管理器支持队列，可以为队列定义不同优 先级或容量限制，这样 Spark 就可以把作业提交到相应的队列中。

## 7.6 集群管理器

Spark可以运行在各种集群管理器上，并通过集群管理器访问集群中的机器。如果只想在一堆机器上运行 Spark，那么自带的**独立模式**是部署该集群最简单的方法。如果有一个需要与别的分布式应用共享的集群(比如既可以运行 Spark 作业又可以运行 Hadoop MapReduce 作业)，Spark 也可以运行在两个广泛使用的集群管理器——Hadoop YARN 与 Apache Mesos 上面。最后，在把 Spark 部署到 Amazon EC2 上时，Spark 有个自带的脚本可以启动独立模式集群以及各种相关服务。本节会介绍如何在这些环境中分别运 行 Spark。

### 7.6.1 独立集群管理器

Spark独立集群管理器是提供在集群上运行的最简单方法，由一个主节点和多个工作节点组成，各自都有分配一定是数量的内存和CPU。提交应用时，可以指定Executor使用内存量，以及所有执行器进程使用的 CPU 核心总数。

**1.启动独立集群管理器**

要启动独立集群管理器，你既可以通过手动启动一个主节点和多个工作节点来实现，也可 以使用 Spark 的 sbin 目录中的启动脚本来实现。启动脚本使用最简单的配置选项，但是需要预先设置机器机器间的SSH无密码访问。

Mac OS X和 Linux系统的部署：

- 将编译好的 Spark 复制到所有机器的一个相同的目录下，比如 /home/yourname/spark。

- 设置好从主节点机器到其他机器的 SSH 无密码登陆。这需要在所有机器上有相同的用 户账号，并在主节点上通过 ssh-keygen 生成 SSH 私钥，然后将这个私钥放到所有工作 节点的 .ssh/authorized_keys 文件中。如果你之前没有设置过这种配置，你可以使用如下 命令：

	```shell
	# 在主节点上:运行ssh-keygen并接受默认选项
	$ ssh-keygen -t dsa
	Enter file in which to save the key (/home/you/.ssh/id_dsa): [回车] Enter passphrase (empty for no passphrase): [空]
	Enter same passphrase again: [空]
	
	# 在工作节点上:
	# 把主节点的~/.ssh/id_dsa.pub文件复制到工作节点上，然后使用: $ cat ~/.ssh/id_dsa.pub >> ~/.ssh/authorized_keys
	$ chmod 644 ~/.ssh/authorized_keys
	```

- 编辑主节点的 conf/slaves 文件并填上所有工作节点的主机名。

- 在**主节点**上运行 sbin/start-all.sh(要在主节点上运行而不是在工作节点上)以启动集群。 如果全部启动成功，你不会得到需要密码的提示符，而且可以在 http://masternode:8080 看到集群管理器的网页用户界面，上面显示着所有的工作节点。

- 要停止集群，在主节点上运行 bin/stop-all.sh。

如果你使用的不是 UNIX 系统（例如：Windows），或者想手动启动集群，你也可以使用 Spark 的 bin/ 目录下的 spark-class 脚本分别手动启动主节点和工作节点。

在主节点上输入:：bin/spark-class org.apache.spark.deploy.master.Master

然后在工作节点上输入：

bin/spark-class org.apache.spark.deploy.worker.Worker spark://masternode:7077 (其中 masternode 是你的主节点的主机名)。在 Windows 中，使用 \ 来代替 /。

默认情况下，集群管理器会选择合适的默认值自动为所有工作节点分配 CPU 核心与内 存。、

**2.提交应用**

要向独立集群管理器提交应用，需要把 spark://masternode:7077 作为主节点参数传给 spark-submit。例如:

```sh
spark-submit --master spark://masternode:7077 yourapp
```

**注意：**这个集群的 URL 也显示在独立集群管理器的网页界面(位于 http://masternode:8080)上。 注意，提交时使用的主机名和端口号必须精确匹配用户界面中的 URL。这有可能会使得使用IP 地址而非主机名的用户遇到问题。即使 IP 地址绑定到同一台主机上，如果名字不是完全匹配的话，提交也会失败。有些管理员可能会配置 Spark 不使用 7077 端口而使用别的端口。**要确保主机名和端口号的一致性，一个简单的方法是从主节点的用户界面中直接复制粘贴 URL。**

你可以使用 --master 参数以同样的方式启动 spark-shell 或 pyspark，来连接到该集群上: 

```shell
spark-shell --master spark://masternode:7077
pyspark --master spark://masternode:7077
```

要检查你的应用或者 shell 是否正在运行，你需要查看集群管理器的网页用户界面 http:// masternode:8080 并确：

(1)应用连接上了(即出现在了 Running Applications 中);

(2) 列出的所使用的核心和内存均大于 0。

**注意：**阻碍应用运行的一个常见陷阱是为执行器进程申请的内存(spark-submit 的 --executor-memory 标记传递的值)超过了集群所能提供的内存总量。在这种情况下，独立集群管理器始终无法为应用分配执行器节点。请确保应用申请的值能够被集群接受。

独立集群管理器支持两种作业提交部署模式：客户端模式（---deploy mode clien）和集群模式（---deploy mode cluster）。

- 客户端模式：驱动器程序会运行在执行spark-submit的机器上，是spark-submit的一部分这意味着你可以直接看到驱动器程序的输出，也 可以直接输入数据进去(通过交互式 shell)，但是这要求你提交应用的机器与工作节点间 有很快的网络速度，并且在程序运行的过程中始终可用。
- 集群模式：驱动器程序会作为某个工作节点上一个独立的进程运行在独立集群管理器内部它也会连接主节点 来申请执行器节点。在这种模式下，spark-submit 是“一劳永逸”型，你可以在应用运行 时关掉你的电脑。你还可以通过集群管理器的网页用户界面访问应用的日志。向 spark- submit 传递 --deploy-mode cluster 参数可以切换到集群模式。

**3.资源配置用量**

在多用户多应用共享Spark集群时，集群管理器需要决定如何在执行器进程间分配资源。独立集群管理器使用基础的调度策略，通过限制各个应用的资源用量让多个应用并发执行。 Apache Mesos 支持应用运行时的更动态的资源共享，而 YARN 则有分级队列的概念，可以限制不同类别的应用的用量。

在独立集群管理器中，资源分配靠下面两个设置来控制：

- 执行器内存

	spark-submit的--executor--memory参数来配置此项。**每个应用在每个工作节点上最多有一个执行器进程**，因此这个配置项能够控制执行器节点占用工作节点的多少内存。此设置项的默认值是 1 GB，在大多数服务器上，可能需要提高这个值来充分利用机器。

- 占用CPU核数总数的最大值

	这是一个应用中所有执行器进程所占用的CPU总数。此项的默认值是无限。也就是说， 应用可以在集群所有可用的节点上启动执行器进程。对于多用户的工作负载来说，应该要求用户限制他们的用量。通过 spark-submit 的 --total-executorcores参数 设置这个值，或者在你的Spark 配置文件中设置 spark.cores.max 的值。

独立集群管理器在默认情况下会为一个应用使用尽可能分散的执行器进程。例如：假设有一个 20 个物理节点的集群，每个物理节点是一个四核的机器，然后使用 --executor-memory 1G 和 --total-executor-cores 8 提交应用。这样 Spark 就会在不同机器上启动 8 个执行器进程，每个 1 GB 内存。Spark 默认这样做，**以尽量实现对于运行在相同机器上的分布式文件系统(比如 HDFS)的数据本地化**，因为这些文件系统通常也把数 据分散到所有物理节点上。如果你愿意，可以通过设置配置属性 spark.deploy.spreadOut 为 false 来要求 Spark 把执行器进程合并到尽量少的工作节点中。在这样的情况下，前面 那个应用就只会得到两个执行器节点，每个有 1 GB 内存和 4 个核心。这一设定会影响运 行在独立模式集群上的所有应用，并且必须在启动独立集群管理器之前设置好。

**4.高可用性**

Spark支持Apache Zookeeper来维护多个备用主节点，保证集群的高可用性（High Available，简称HA）。

### 7.6.2 Hadoop YARN

YARN 是在 Hadoop 2.0 中引入的集群管理器，它可以让多种数据处理框架运行在一个共享的资源池上，并且通常安装在与 Hadoop 文件系统(简称 HDFS)相同的物理节点上。在 这样配置的 YARN 集群上运行 Spark 是很有意义的，它可以让 Spark 在存储数据的物理节 点上运行，以快速访问 HDFS 中的数据。

在 Spark 里使用 YARN 很简单:你只需要设置指向你的 Hadoop 配置目录的环境变量，然 后使用 spark-submit 向一个特殊的主节点 URL 提交作业即可。具体步骤如下：

- 找到Hadoop 的配置目录，并把它设为环境变量 HADOOP_CONF_DIR。这个目录 包含 yarn-site.xml 和其他配置文件;

- 如下方式提交应用。

	```sh
	export HADOOP_CONF_DIR="..."
	spark-submit --master yarn yourapp
	```

和独立集群管理器一样，有两种将应用连接到集群的模式：客户端模式以及集群模式。在客户端模式下应用的驱动器程序运行在提交应用的机器上(比如你的笔记本电脑)，而在集群模式下，驱动器程序也运行在一个YARN容器内部。你可以通过 spark-submit 的--deploy-mode：client和cluster参数设置不同的模式。

Spark 的交互式 shell 以及 pyspark 也都可以运行在 YARN 上。只要设置好 HADOOP_CONF_ DIR 并对这些应用使用 --master yarn 参数即可。注意，由于这些应用需要从用户处获取输 入，所以只能运行于客户端模式下。

### 7.6.3 Apache Mesos

Apache Mesos是一个通用的集群管理器，既可以运行分析型工作负载又可以运行长期运行的服务，比如网页服务或键值对存储服务。要在 Mesos 上使用 Spark，需要把一个 mesos:// 的 URI 传给 spark-submit:

```sh
spark-submit --master mesos://masternode:5050 yourapp
```

在运行多个主节点时，可以使用ZooKeeper为Mesos集群选出一个主节点，在这种情况下，应该使用 mesos://zk:// 的 URI 来指向一个 ZooKeeper 节点列表。比如，有三个 ZooKeeper 节点(node1、node2 和 node3)，并且 ZooKeeper 分别运行在三台机器的 2181 端 口上时，你应该使用如下 URI:

```sh
mesos://zk://node1:2181/mesos,node2:2181/mesos,node3:2181/mesos
```

**1.Mesos调度模式**

Mesos 提供了两种模式来在一个集群内的执行器进程间共享资源——细粒度模式与粗粒度模式，默认使用细粒度模式。

- 细粒度模式：执行器占用的CPU核心数会在执行任务时动态变化，因此一台运行了多个执行器进程的机器可以共享CPU资源。
- 粗粒度模式：Spark提前为各个执行器分配固定数量的CPU数目，并在应用结束前不会释放这些资源，即使Executor处于空闲状态。可以通过向spark-submit 传递 --conf spark.mesos.coarse=true 来打开粗粒度模式。

当多用户共享的集群运行 shell 这样的交互式的工作负载时，由于应用会在它们不工作时降低它们所占用的核心数，以允许别的用户程序使用集群，所以这种情况下细粒度模式显得非常合适。然而，在细粒度模式下调度任务会带来更多的延迟(一些像 Spark Streaming这样需要低延迟的应用就会表现很差)，应用需要在用户输入新的命令时，为重新分配CPU核心等待一段时间。可以在一个 Mesos 集群中使用**混合的调度模式**(比如将一部分 Spark 应用的 spark.mesos.coarse 设置为 true，而另一部分 不这么设置)。

**2.客户端与集群模式**

在Spark2，在 Mesos 上 Spark 支持以客户端和集群的部署模式运行应用。

- 客户端模式：客户端机器上将会启动一个Spark Mesos框架，并且会等待驱动（driver）的输出。驱动需要spark-env.sh中的一些配置项，以便和Mesos交互操作：

	```sh
	export MESOS_NATIVE_JAVA_LIBRARY=<path to libmesos.so>
	export SPARK_EXECUTOR_URI=<上文所述的上传spark-1.6.0.tar.gz对应的URL>
	```

- 集群模式：驱动器将在集群中某一台机器上启动，其运行结果可以在Mesos Web UI上看到。要使用集群模式，你首先需要利用 sbin/start-mesos-dispatcher.sh脚本启动 MesosClusterDispatcher，并且将Mesos Master URL（如：mesos://host:5050）传给该脚本。MesosClusterDispatcher启动后会以后台服务的形式运行在本机。Mesos集群提交任务如下示例：

	```sh
	./bin/spark-submit \
	  --class org.apache.spark.examples.SparkPi \
	  --master mesos://207.184.161.138:7077 \
	  --deploy-mode cluster \
	  --supervise \
	  --executor-memory 20G \
	  --total-executor-cores 100\
	  http://path/to/examples.jar \
	  1000
	```

### 7.6.4 Amazon EC2

Spark 自带一个可以在 Amazon EC2 上启动集群的脚本。这个脚本会启动一些节点，并且 在它们上面安装独立集群管理器。

## 7.7 集群选择

Spark 所支持的各种集群管理器提供了部署应用的多种选择。如果需要从零开始部署，正在权衡各种集群管理器，推荐如下一些准则：

- 如果是从零开始，可以先选择独立集群管理器。独立模式安装起来最简单，而且如果只是使用 Spark 的话，独立集群管理器提供与其他集群管理器完全一样的全部功能。
- 如果你要在使用 Spark 的同时使用其他应用，如：HDFS，ZooKeeper，或者是要用到更丰富的资源调度功能(例如队列)，那么 YARN 和 Mesos 都能满足需求。而在这两者中，对于大多数 Hadoop 发行版来说，一般 YARN 已经预装好了。
- Mesos 相对于 YARN 和独立模式的一大优点在于其**细粒度共享**的选项，该选项可以将 类似 Spark shell 这样的交互式应用中的不同命令分配到不同的 CPU 上。因此这对于多 用户同时运行交互式 shell 的用例更有用处。
- 任何情况下，最好把Spark运行在HDFS节点上，这样能快速访问存储，提高运行速度，减少IO交互，可以自行在同样节点上安装Mesos或独立集群管理器，如果使用 YARN 的话，大多数发 行版已经把 YARN 和 HDFS 安装在了一起。

最后可以通过查阅所使用 Spark 版本的官方文档(http://spark.apache.org/docs/latest/)来了解集群管理器最新的选项。

## 7.8 总结

本章描述了：

- Spark应用的运行时框架，由一个驱动器节点和一系列分布式执行器节点组成；
- 如何构建，打包Spark应用，并向集群提交执行；
- 常见的Spark集群部署环境：独立集群管理器，YARN，Mesos，Amazon EC2 云服务上运行。

