# 第九章 Spark SQL

本章介绍Spark用于操作结构化和半结构化数据的接口——Spark SQL。结构化数据指任何有结构信息的数据，所谓结构信息，就是每条记录共用的已知字段集合。

Spark SQL 提供了以下三大功能：

- Spark SQL可以从各个结构化数据源（例如：JSON，Hive，Parquet等）中读取数据；
- )Spark SQL 不仅支持在 Spark 程序内使用 SQL 语句进行数据查询，也支持从类似商业 智能软件 Tableau 这样的外部工具中通过标准数据库连接器(JDBC/ODBC)连接 Spark SQL 进行查询。
- 在Spark程序内使用Spark SQL时，Spark SQL支持SQL与常规的Python/Java/Scala代码高度整合，包括连接RDD和SQL表，公开的自定义SQL函数接口等。

为了实现这些功能Spark提供了一种特殊的RDD，叫做**SchemaRDD**。SchemaRDD是存放Row对象的RDD，每个Row代表一行记录。SchemaRDD 还包含记录的结构信 息(即数据字段)。SchemaRDD 支持 RDD 上所没有的一些新操作，比如运行 SQL 查询。SchemaRDD 可以从外部数据源创建，也可以从查询结果或普通 RDD 中创建。

![](./img/9-1.jpg)

​																	**图9-1 Spark SQL的用途**

本章会先讲解如何在常规 Spark 程序中使用 SchemaRDD，以读取和查询结构化数据。接下 来会讲解 Spark SQL 的 JDBC 服务器，它可以让你在一个共享的服务器上运行 Spark SQL， 也可以让 SQL shell 或者类似 Tableau 的可视化工具连接它而使用。最后会讨论更多高级特 性。Spark SQL 是 Spark 中比较新的组件，在 Spark 1.3 以及后续版本中还会有重大升级， 因此要想获取关于 Spark SQL 和 SchemaRDD 的最新信息，请访问最新版本的文档。

## 9.1 连接Spark SQL

Apache Hive是Hadoop上的SQL引擎，Spark SQL编译时可以包含Hive支持，也可以不包含。包含 Hive 支持的 Spark SQL 可以支持 Hive 表访问、UDF(用户自定义函数)、 SerDe(序列化格式和反序列化格式)，以及Hive查询语言(HiveQL/HQL)。需要强调的 一点是，如果要在 Spark SQL 中包含 Hive 的库，并不需要事先安装 Hive。一般来说，**最好还是在编译 Spark SQL 时引入 Hive 支持**，这样就可以使用这些特性了。如果下载的是二进制版本的 Spark，它应该已经在编译时添加了 Hive 支持。而如果你是从代码编译 Spark，你应该使用 sbt/sbt -Phive assembly 编译，以打开 Hive 支持。

在 Java 以及 Scala 中，连接带有 Hive 支持的 Spark SQL 的 Maven 索引如例 9-1 所示。 

**例 9-1:带有 Hive 支持的 Spark SQL 的 Maven 索引**

```yml
      groupId = org.apache.spark
      artifactId = spark-hive_2.10
      version = 1.2.0
```

如果你不能引入 Hive 依赖，那就应该使用工件 spark-sql_2.10 来代替 spark-hive_2.10。

当使用 Spark SQL 进行编程时，根据是否使用 Hive 支持，有两个不同的入口。推荐使用的入口是 **HiveContext**，它可以提供 HiveQL 以及其他依赖于 Hive 的功能的支持。更为基础的 SQLContext 则支持 Spark SQL 功能的一个子集，子集中去掉了需要依赖于 Hive 的功 能。这种分离主要是为那些可能会因为引入 Hive 的全部依赖而陷入依赖冲突的用户而设 计的。**使用 HiveContext 不需要事先部署好 Hive**。

若要把 Spark SQL 连接到一个部署好的 Hive 上，你必须把 hive-site.xml复制到 Spark 的配置文件目录中($SPARK_HOME/conf)。即使没有部署好 Hive，Spark SQL也可以运行。

**Tips**

如果没有部署Hive，Spark SQL会在当前的工作目录中创建出自己的Hive元数据仓库，叫作metastore_db。此外，如果你尝试使用 HiveQL 中的 CREATE TABLE (并非 CREATE EXTERNAL TABLE)语句来创建表，这些表会被放在你默认的文件系统中的 /user/hive/warehouse 目录中(如果你的 classpath 中有配好的 hdfs-site.xml，默认的文件系统就是 HDFS，否则就是本地文件系统)。

## 9.2 在应用中使用Spark SQL

Spark SQL 最强大之处就是可以在 Spark 应用内使用。这种方式可以轻松读取数据并使用 SQL 查询，同时还能把这一过程和普通的 Python/Java/Scala 程序代码结合在一起。

需要基于已有的SparkContext创建出一个HiveContext，HiveContext上下文环境提供了对Spark SQL的数据进行查询和交互的额外函数，可以创建出表示结构化数据 的 SchemaRDD，并且使用 SQL 或是类似 map() 的普通 RDD 操作来操作这些 SchemaRDD。

### 9.2.1 初始化Spark SQL

- 导入Spark SQL相关模块：

**例 9-4:Java 中 SQL 的 import 声明**

```java
// 导入Spark SQL
import org.apache.spark.sql.hive.HiveContext; // 当不能使用hive依赖时
import org.apache.spark.sql.SQLContext;
// 导入JavaSchemaRDD
import org.apache.spark.sql.SchemaRDD;
import org.apache.spark.sql.Row;
```

**例 9-5:Python 中 SQL 的 import 声明**

```python
# 导入Spark SQL
from pyspark.sql import HiveContext, Row 
# 当不能引入hive依赖时
from pyspark.sql import SQLContext, Row
```

- 创建HiveContext对象

	例 9-6:在 Scala 中创建 SQL 上下文环境 

	```scala
	val sc = new SparkContext(...)
	val hiveCtx = new HiveContext(sc)
	```

	例 9-7:在 Java 中创建 SQL 上下文环境

	```java
	JavaSparkContext ctx = new JavaSparkContext(...);
	SQLContext sqlCtx = new HiveContext(ctx);
	```

	例 9-8:在 Python 中创建 SQL 上下文环境 

	```python
	hiveCtx = HiveContext(sc)
	```

### 9.2.2 基本查询示例

需要调用HiveContext和SQLContext中的sql( )方法。要做的第一件事就是告诉 Spark SQL 要查询的数据是什么。例如：JSON文件：先从 JSON 文件中读取数据，把这些数据注册为一张临时表并赋予该表一个名字，然后就可以用 SQL 来查询它了。

**例 9-9:在 Scala 中读取并查询推文**

```scala
val input = hiveCtx.jsonFile(inputFile)
 // 注册输入的SchemaRDD
input.registerTempTable("tweets")
// 依据retweetCount(转发计数)选出推文
 val topTweets = hiveCtx.sql("SELECT text, retweetCount FROMtweets ORDER BY retweetCount LIMIT 10")
```

**例 9-10:在 Java 中读取并查询推文**

```java
SchemaRDD input = hiveCtx.jsonFile(inputFile);
// 注册输入的SchemaRDD
input.registerTempTable("tweets");
// 依据retweetCount(转发计数)选出推文
SchemaRDD topTweets = hiveCtx.sql("SELECT text, retweetCount FROM
       tweets ORDER BY retweetCount LIMIT 10");
```

**例 9-11:在 Python 中读取并查询推文**

```python
input = hiveCtx.jsonFile(inputFile)
# 注册输入的SchemaRDD
input.registerTempTable("tweets")
# 依据retweetCount(转发计数)选出推文
topTweets = hiveCtx.sql("""SELECT text, retweetCount FROM
       tweets ORDER BY retweetCount LIMIT 10""")
```

**Tips**

如果已经有安装好的Hive，并且已经把 hive-site.xml 文件复制到了$SPARK_HOME/conf 目录下，那么也可以直接运行 hiveCtx.sql 来查询已 有的 Hive 表。此外书中Spark版本较低，上面的方法在Spark SQL中均不再使用。我们可以了解其实现思想。

### 9.2.3 SchemaRDD

读取数据和执行查询都会返回 SchemaRDD。SchemaRDD 和传统数据库中的表的概念类 似。从内部机理来看，SchemaRDD 是一个由 Row 对象组成的 RDD，附带包含每列数据类 型的结构信息。Row 对象只是对基本数据类型(如整型和字符串型等)的数组的封装。

需要特别注意的是，在今后的 Spark 版本中(1.3 及以后)，SchemaRDD被改为 **DataSet< Row >**，详细情况请见官方文档。

SchemaRDD 可以对其应用已有的 RDD 转化操作，比如 map() 和 filter()。然而，SchemaRDD 也提供了一些额外的功能支持。最重要的是，可以把任意SchemaRDD注册为临时表(registerTempTable)，这样就可以使用 HiveContext.sql 或 SQLContext.sql 来对进行查询了。

**Tips 临时表**

临时表是当前使用的 HiveContext 或 SQLContext 中的临时变量，在应用退出时这些临时表就不再存在了。

SchemaRDD 可以存储一些基本数据类型，也可以存储由这些类型组成的结构体和数组。 SchemaRDD使用 HiveQL 语法定义的类型。表 9-1 列出了支持的数据类型。

**表9-1:SchemaRDD中可以存储的数据类型**

| HiveSql/SparkSql类型      | Scala类型             | Java类型             | Python类型                         |
| ------------------------- | --------------------- | -------------------- | ---------------------------------- |
| TINYINT                   | Byte                  | Byte/byte            | int/long ( 在 -128 到 127 之间 )   |
| SMALLINT                  | Short                 | Short/short          | int/long ( 在 -32768 到 32767之间) |
| INT                       | int                   | Int/Integer          | int 或 long                        |
| BIGINT                    | long                  | Long/Long            | Long                               |
| FLOAT                     | Float                 | Float/Float          | Float                              |
| DOUBLE                    | double                | Double/Double        | Float                              |
| DECIMAL                   | Scala.math.BigDecimal | java.math.BigDecimal | decimal.Decimal                    |
| STRING                    | String                | String               | string                             |
| BINARY                    | Array[Byte]           | byte[]               | bytearray                          |
| TIMESTAMP                 | java.sql.TimeStamp    | java.sql.TimeStamp   | Datatime.datatime                  |
| BOOLEAN                   | Boolean               | Boolean/boolean      | bool                               |
| ARRAY<DATA_TYPE>          | Sql                   | List                 | list、tuple 或 array               |
| MAP<KEY_TYPE, VALUE_TYPE> | Map                   | Map                  | Dict                               |
| STRUCT<COL1:COL1_TYPE >   | Row                   | Row                  | Row                                |

**使用Row对象**

Row 对象表示 SchemaRDD 中的记录，其本质就是一个定长的字段数组。在 Scala/Java 中， Row 对象有一系列 getter 方法，可以通过下标获取每个字段的值。标准的取值方法 get(或 Scala 中的 apply)，读入一个列的序号然后返回一个 Object 类型(或 Scala 中的 Any 类型) 的对象，然后由我们把对象转为正确的类型。对于 Boolean、Byte、Double、Float、Int、 Long、Short 和 String 类型，都有对应的 getType() 方法，可以把值直接作为相应的类型 返回。例如，getString(0) 会把字段 0 的值作为字符串返回，如例 9-12 和例 9-13 所示。

**例 9-12:在 Scala 中访问 topTweet 这个 SchemaRDD 中的 text 列(也就是第一列)** 

```scala
val topTweetText = topTweets.map(row => row.getString(0)
```

**例 9-13:在 Java 中访问 topTweet 这个 SchemaRDD 中的 text 列(也就是第一列)**

```java
JavaRDD<String> topTweetText = topTweets.toJavaRDD().map(new Function<Row, String>() { public String call(Row row) {
           return row.getString(0);
         }});
```

在 Python 中，由于没有显式的类型系统，Row 对象变得稍有不同。使用 row[i] 来访问第 *i* 个元素。除此之外，还支持以 row*.column_name*的形式使用名字来访问其中的字段：

**例 9-14:在 Python 中访问 topTweet 这个 SchemaRDD 中的 text 列** 

```python
topTweetText = topTweets.map(lambda row: row.text)
```

### 9.2.4 缓存

Spark SQL的缓存机制与Spark中的稍有不同。由于我们知道每个列的类型信息，所以 Spark可以更加高效地存储数据。为了确保使用更节约内存的表示方式进行缓存而不是储存整个对象，应当使用专门的 **hiveCtx.cacheTable("tableName")** 方法。当缓存数据表时， Spark SQL使用一种列式存储格式在内存中表示数据。这些缓存下来的表只会在驱动器程序的生命周期里保留在内存中，所以如果驱动器进程退出，就需要重新缓存数据。和缓存 RDD 时的动机一样，如果想在同样的数据上多次运行任务或查询时，就应把这些数据表缓存起来。

被缓存的 SchemaRDD 以与其他 RDD 相似的方式在 Spark 的应用用户界面中呈现，