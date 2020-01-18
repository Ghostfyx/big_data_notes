# 2.3 Spark核心概念简介

每个Spark应用都由一个驱动器程序（driver program）来发起集群上的各种并行操作，驱动器程序包含应用的main函数，如：

```java
// Java
public static void main(String[] args){}
```

```python
if __name == '__main__':
```

```scala
def main(args:Array[String]) {  
       
    }  
```

并且定义了集群上的分布式数据集RDD，还对分布式数据集应用了相关操作。

驱动器通过SparkContext对象来访问Spark，这个对象代表计算机集群的一个连接，当我们在命令行启动spark-shell的时候就自动创建了一个SparkContext对象，叫做sc，如下所示：

```shell
localhost  /  Spark-shell --master=local
20/01/19 05:21:27 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Using Spark's default log4j profile: org/apache/spark/log4j-defaults.properties
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
Spark context Web UI available at http://192.168.1.3:4040
Spark context available as 'sc' (master = local, app id = local-1579382493361).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.4.4
      /_/

Using Scala version 2.11.12 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_202)
Type in expressions to have them evaluated.
Type :help for more information.

scala> sc
res0: org.apache.spark.SparkContext = org.apache.spark.SparkContext@1d0acb8f
```

 如果我们安装了pyspark，我们可以在命令行启动pyspark：

```shell
localhost  /  pyspark
Python 3.6.3 (v3.6.3:2c5fed86e0, Oct  3 2017, 00:32:08)
[GCC 4.2.1 (Apple Inc. build 5666) (dot 3)] on darwin
Type "help", "copyright", "credits" or "license" for more information.
20/01/19 06:08:14 WARN NativeCodeLoader: Unable to load native-hadoop library for your platform... using builtin-java classes where applicable
Using Spark's default log4j profile: org/apache/spark/log4j-defaults.properties
Setting default log level to "WARN".
To adjust logging level use sc.setLogLevel(newLevel). For SparkR, use setLogLevel(newLevel).
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /__ / .__/\_,_/_/ /_/\_\   version 2.4.4
      /_/

Using Python version 3.6.3 (v3.6.3:2c5fed86e0, Oct  3 2017 00:32:08)
SparkSession available as 'spark'.
>>>
```

我们可以用SparkContext创建RDD，并对RDD进行一系列操作，要执行这些操作，驱动器程序一般要管理多个执行器（executor）节点，不同的节点对执行分布式数据集RDD不同部分的数据，已达到运算并行化的目的。

![](https://img-blog.csdn.net/20181011090802244)

Spark对操作抽象了许多高级API方法，可以用Python，Scala和Java实现，具体实现中推荐使用Lambda表达式形式，代码简介易读。

# 2.4 独立应用

本节介绍如何在独立应用中使用Spark，在Python，Java和Scala的独立应用中被连接使用，与Shell命令的主要区别是需要创建SparkContext。

Java和Scala中，引入Maven的Spark-core依赖。在Python中，可以把应用写成python脚本，使用Spark自带的bin/spark-submit脚本来运行，spark-submit会自动帮我们引入Python程序的Spark依赖。

```shell
bin/spark-submit xxxx.py
```

### 2.4.1 初始化SparkContext

使用Python：

```python
from pyspark import SparkConf, SparkContext

conf = SparkConf().setMaster("local").setAppName("Spark-init")
sc = SparkContext(conf=conf)
print(sc)
```

使用Java

```java
import org.apache.spark.api.java.JavaSparkContext;
import org.apache.spark.SparkConf;

SparkConf sparkConf = new SparkConf().setMaster("local").setAppName("Spark-init");
SparkContext sc = new SparkContext(sparkConf)
```

最简单的方法只需要传递两个参数：

- Spark集群URL
- 应用名

最后，关闭Spark可以调用SparkContext的stop方法，或者直接退出应用，如sys.exit()或 System.exit(0) 。

### 2.4.2 构建独立应用

Java可以使用Maven将项目打包为Jar，运行spark-submit命令执行。python直接上传脚本文件即可。

```bash
#!/bin/bash
# Java方式的项目运行脚本
export JAVA_HOME=/usr/local/java/jdk1.8.0_231
echo "JAVA_HOME=$JAVA_HOME"
export BOOK_HOME=/opt/data_algorithms_book
export APP_JAR=/opt/data_algorithms_book/leftjoin/chap04-1.0-SNAPSHOT.jar
export HADOOP_HOME=/opt/hadoop/hadoop-2.7.7
export HADOOP_CONF=$HADOOP_HOME/etc/hadoop
export YARN_CONF=$HADOOP_HOME/etc/hadoop
export spark_home=/opt/spark/spark-2.4.4-bin-hadoop2.7
USERS=/leftjoin/input/users.txt
TRANSACTIONS=/leftjoin/input/transactions.txt
OUTPUT=/leftjoin/output/spark
prog=org.dataalgorithms.leftjoin.spark.SparkUseLeftJoin
$SPARK_HOME/bin/spark-submit \
 --class $prog \
 --master yarn \
 --deploy-mode client \
 --executor-memory 2G\
 --num-executors 10 \
 $APP_JAR \
 $USERS \
 $TRANSACTIONS \
 $OUTPUT

```

```shell
# For Scala and Java, use run-example:
./bin/run-example SparkPi

# For Python examples, use spark-submit directly:
./bin/spark-submit examples/src/main/python/pi.py

# For R examples, use spark-submit directly:
./bin/spark-submit examples/src/main/r/dataframe.R
```

