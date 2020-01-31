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

