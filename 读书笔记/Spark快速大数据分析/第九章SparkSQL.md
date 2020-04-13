

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

## 9.3  读取数据

Spark SQL 支持多种结构化数据源，可以跳过复杂的读取过程，轻松从各种数据源中读取到 Row 对象。这些数据源包括 Hive 表、JSON 和 Parquet 文件。此外，当使用 SQL 查询这些数据源中的数据并且只用到了一部分字段时，Spark SQL 可以智能地只扫描这些用到的字段，而不是像 SparkContext.hadoopFile 中那样简单粗暴地扫描全部数据。

除这些数据源之外，也可以在程序中通过指定结构信息，将常规的 RDD 转化为SchemaRDD。这使得在 Python 或者 Java 对象上运行 SQL 查询更加简单。当需要计算许多数值时，SQL 查询往往更加简洁(比如要同时求出平均年龄、最大年龄、不重复的用 户 ID 数目等)。不仅如此，还可以自如地将这些 RDD 和来自其他 Spark SQL 数据源的 SchemaRDD 进行连接操作。在本节中，会讲解外部数据源以及这种使用 RDD 的方式。

### 9.3.1 Apache Hive

从Hive中读取数据时，Spark SQL支持任何Hive支持的存储格式（SerDe），包括：文本文件、RCFiles、ORC、Parquet、Avro以及 Protocol Buffer。

要把 Spark SQL 连接到已经部署好的Hive上，只需要把Hive的hive-site.xml文件复制到 Spark 的 ./conf/ 目录下即可。如果只是想探索一下 Spark SQL 而没有配置 hive-site.xml 文件，那么 Spark SQL 则会使用本地的 Hive 元数据仓，并且同样可以轻松地将数据读取到 Hive 表中进行查询。

**例 9-15:使用 Python 从 Hive 读取**

```python
from pyspark.sql import HiveContext
hiveCtx = HiveContext(sc)
rows = hiveCtx.sql("SELECT key, value FROM mytable")
keys = rows.map(lambda row: row[0])
```

**例 9-16:使用 Scala 从 Hive 读取**

```scala
import org.apache.spark.sql.hive.HiveContext
val hiveCtx = new HiveContext(sc)
val rows = hiveCtx.sql("SELECT key, value FROM mytable")
val keys = rows.map(row => row.getInt(0))
```

**例 9-17:使用 Java 从 Hive 读取**

```java
import org.apache.spark.sql.hive.HiveContext;
import org.apache.spark.sql.Row;
import org.apache.spark.sql.SchemaRDD;
 HiveContext hiveCtx = new HiveContext(sc);
     SchemaRDD rows = hiveCtx.sql("SELECT key, value FROM mytable");
     JavaRDD<Integer> keys = rdd.toJavaRDD().map(new Function<Row, Integer>() {
       public Integer call(Row row) { return row.getInt(0); }
     });
```

### 9.3.2 Parquet

Parquet是一种流行的列存储格式，可以高效地存储具有嵌套字段的记录。Parquet 格式经常在 Hadoop 生态圈中被使用，它也支持 Spark SQL 的全部数 据类型。Spark SQL 提供了直接读取和存储 Parquet 格式文件的方法。

可以通过 HiveContext.parquetFile 或者 SQLContext.parquetFile 来读取数据，如例 9-18 所示。

例 9-18:Python 中的 Parquet 数据读取

```python
# 从一个有name和favouriteAnimal字段的Parquet文件中读取数据 
rows = hiveCtx.parquetFile(parquetFile)
names = rows.map(lambda row: row.name)
print("Everyone")
print(names.collect())
```

也可以把 Parquet 文件**注册为 Spark SQL 的临时表**，并在这张表上运行查询语句。在例 9-18 中我们读取了数据，接下来可以参照例 9-19 所示的对数据进行查询。

**例 9-19:Python 中的 Parquet 数据查询**

```python
# 寻找熊猫爱好者
tbl = rows.registerTempTable("people")
pandaFriends = hiveCtx.sql("SELECT name FROM people WHERE favouriteAnimal = \"panda\"")
print "Panda friends"
print pandaFriends.map(lambda row: row.name).collect()
```

可以使用 saveAsParquetFile() 把 SchemaRDD 的内容以 Parquet 格式保存，如例 9-20 所示。

**例 9-20:Python 中的 Parquet 文件保存** 

```python
pandaFriends.saveAsParquetFile("hdfs://...")
```

### 9.3.3 JSON

如果有一个 JSON 文件，其中的记录遵循同样的结构信息，那么Spark SQL 就可以通过扫描文件推测出结构信息，并且可以使用名字访问对应字段(如例 9-21 所示)。如果在一个包含大量 JSON 文件的目录中进行尝试，会发现Spark SQL的结构信息推断可以让非常高效地操作数据，而无需编写专门的代码来读取不同结构的文件。

要读取 JSON 数据，只要调用 hiveCtx 中的 jsonFile() 方法即可，如例 9-22 至例 9-24 所示。如果你想获得从数据中推断出来的结构信息，可以在生成的 SchemaRDD 上调用 printSchema 方法(见例 9-25)。

例 9-21:输入记录 

```json
{"name": "Holden"}
{"name": "Sparky The Bear", "lovesPandas":true,"knows": {"friends":["holden"]}}
```

例 9-22:在 Python 中使用 Spark SQL 读取 JSON 数据 

```scala
input = hiveCtx.jsonFile(inputFile)
```

例 9-23:在 Scala 中使用 Spark SQL 读取 JSON 数据 

```scala
val input = hiveCtx.jsonFile(inputFile)
```

例 9-24:在 Java 中使用 Spark SQL 读取 JSON 数据 

```scala
SchemaRDD input = hiveCtx.jsonFile(jsonFile);
```

### 9.3.4 基于RDD

除了读取数据，也可以基于 RDD 创建 SchemaRDD。在 Scala 中，带有 case class 的 RDD可以隐式转换成 SchemaRDD。在 Python 中，可以创建一个由 Row 对象组成的 RDD，然后调用 inferSchema()，如例 9-28所示。
 	**例 9-28:在 Python 中使用 Row 和具名元组创建 SchemaRDD**

```python
happyPeopleRDD = sc.parallelize([Row(name="holden", favouriteBeverage="coffee")])
happyPeopleSchemaRDD = hiveCtx.inferSchema(happyPeopleRDD)
happyPeopleSchemaRDD.registerTempTable("happy_people")
```

**例 9-29:在 Scala 中基于 case class 创建 SchemaRDD**

```scala
case class HappyPerson(handle: String, favouriteBeverage: String)
 ...
// 创建了一个人的对象，并且把它转成SchemaRDD
val happyPeopleRDD = sc.parallelize(List(HappyPerson("holden", "coffee")))
// 注意:此处发生了隐式转换
// 该转换等价于sqlCtx.createSchemaRDD(happyPeopleRDD) 
happyPeopleRDD.registerTempTable("happy_people")
```

在 Java 中，可以调用 applySchema() 把 RDD 转为 SchemaRDD，只需要这个 RDD 中的数 据类型带有公有的 getter 和 setter 方法，并且可以被序列化，如例 9-30 所示。

例 9-30:在 Java 中基于 JavaBean 创建 SchemaRDD

```java
     class HappyPerson implements Serializable {
       private String name;
       private String favouriteBeverage;
       public HappyPerson() {}
       public HappyPerson(String n, String b) {
         name = n; favouriteBeverage = b;
       }
       public String getName() { return name; }
       public void setName(String n) { name = n; }
       public String getFavouriteBeverage() { return favouriteBeverage; }
       public void setFavouriteBeverage(String b) { favouriteBeverage = b; }
     };
     ...
     ArrayList<HappyPerson> peopleList = new ArrayList<HappyPerson>();
     peopleList.add(new HappyPerson("holden", "coffee"));
     JavaRDD<HappyPerson> happyPeopleRDD = sc.parallelize(peopleList);
     SchemaRDD happyPeopleSchemaRDD = hiveCtx.applySchema(happyPeopleRDD,
       HappyPerson.class);
     happyPeopleSchemaRDD.registerTempTable("happy_people");
```

## 9.4 JDBC/ODBC服务器

Spark SQL也提供了JDBC连接支持，这对于让商业智能(BI)工具连接到 Spark 集群上以 及在多用户间共享一个集群的场景都非常有用。JDBC 服务器作为一个独立的 Spark驱动器程序运行，可以在多用户之间共享。任意一个客户端都可以在内存中缓存数据表，对表 进行查询。集群的资源以及缓存数据都在所有用户之间共享。

Spark SQL 的 JDBC 服务器与 Hive 中的 HiveServer2 相一致。由于使用了 Thrift 通信协议，它也 、被称为“Thrift server”。注意，JDBC 服务器支持需要 Spark 在打开 Hive 支持的选项下编译。

服务器可以通过 Spark 目录中的 sbin/start-thriftserver.sh 启动(见例 9-31)。这个 脚本接受的参数选项大多与 spark-submit 相同(见 7.3 节)。默认情况下，服务器会在 localhost:10000 上 进 行 监 听， 可以通过环境变量(HIVE_SERVER2_THRIFT_PORT 和 HIVE_SERVER2_THRIFT_BIND_HOST) 修 改 这 些 设 置， 也 可 以 通 过 Hive 配置选项(hive. server2.thrift.port 和 hive.server2.thrift.bind.host)来修改。也可以通过命令行参数 --hiveconf property=value 来设置 Hive 选项。

例 9-31:启动 JDBC 服务器 

```sh
./sbin/start-thriftserver.sh --master sparkMaster
```

Spark 也自带了Beeline客户端程序，我们可以使用它连接 JDBC 服务器，如例 9-32 和图 9-3 所示。这个简易的 SQL shell 可以让我们在服务器上运行命令。

**例 9-32:使用 Beeline 连接 JDBC 服务器**

```
holden@hmbp2:~/repos/spark$ ./bin/beeline -u jdbc:hive2://localhost:10000
     Spark assembly has been built with Hive, including Datanucleus jars on classpath
     scan complete in 1ms
     Connecting to jdbc:hive2://localhost:10000
     Connected to: Spark SQL (version 1.2.0-SNAPSHOT)
     Driver: spark-assembly (version 1.2.0-SNAPSHOT)
     Transaction isolation: TRANSACTION_REPEATABLE_READ
     Beeline version 1.2.0-SNAPSHOT by Apache Hive
     0: jdbc:hive2://localhost:10000> show tables;
     +---------+
     | result  |
     +---------+
     | pokes   |
     +---------+
     1 row selected (1.182 seconds)
     0: jdbc:hive2://localhost:10000>
```

当启动 JDBC 服务器时，JDBC 服务器会在后台运行并且将所有输出重定向 到一个日志文件中。如果你在使用 JDBC 服务器进行查询的过程中遇到了问 题，可以查看日志寻找更为完整的报错信息。

### 9.4.1 使用Beeline

在Beeline客户端中，可以使用标准的HiveSQL命令创建、列举以及查询数据库表。

**例 9-33:读取数据表**

创建一张数据表，可以使用 CREATE TABLE 命令。然后使用 LOAD DATA 命令进行数据读取。

```sh
> CREATE TABLE IF NOT EXISTS mytable (key INT, value STRING) ROW FORMAT DELIMITED FIELDS TERMINATED BY‘,’;

> LOAD DATA LOCAL INPATH‘learning-spark-examples/files/int_string.csv’ INTO TABLE mytable;
```

使用SHOW TABLES 语句(如例 9-34 所示)。你也可以通过DESCRIBE *tableName*查看每张表的结构信息。

**例 9-34:列举数据表**

```sh
> SHOW TABLES;
	mytable
	Time taken: 0.052 seconds
```

使用CACHE TABLE *tableName*语句，缓存数据库表。缓存之后你可以使用UNCACHE TABLE *tableName* 命令取消对表的缓存。

在Beeline中查看查询计划很简单，对查询语句运行 EXPLAIN 即可，如例 9-35 所示。

**例 9-35:Spark SQL shell 执行 EXPLAIN**

```sh
spark-sql> EXPLAIN SELECT * FROM mytable where key = 1;
    == Physical Plan ==
    Filter (key#16 = 1)
    HiveTableScan [key#16,value#17], (MetastoreRelation default, mytable, None), None
    Time taken: 0.551 seconds
```

### 9.4.2 长生命周期的表与查询

使用 Spark SQL 的 JDBC 服务器的优点之一就是可以在多个不同程序之间共享缓存下 来的数据表。JDBC Thrift 服务器是一个单驱动器程序，这就使得共享成为了可能。如前一 节中所述，你只需要注册该数据表并对其运行 CACHE 命令，就可以利用缓存了。

## 9.5 用户自定义函数

可以让我们使用 Python/Java/Scala 注册自定义函数，并在 SQL 中调用。这种方法很常用，通常用来给机构内的 SQL 用户们提供高级功能支持。在 Spark SQL 中，编写 UDF 尤为简单。Spark SQL 不仅有自己的 UDF 接口，也支持已有的 Apache Hive UDF。

### 9.5.1 Spark SQL UDF

可以使用 Spark 支持的编程语言编写好函数，然后通过 Spark SQL 内建的方法传递进来，非常便捷地注册我们自己的 UDF。在 Scala 和 Python 中，可以利用语言原生的函数和 lambda 语法的支持，而在 Java 中，则需要扩展对应的 UDF 类。UDF 能够支持各种数据类型，返回类型也可以与调用时的参数类型完全不一样。

在例 9-36 和例 9-37 中，我们可以看到一个用来计算字符串长度的非常简易的 UDF，可以 用它来计算推文的长度。

例 9-36:Python 版本耳朵字符串长度 UDF

```python
# 写一个求字符串长度的UDF
hiveCtx.registerFunction("strLenPython", lambda x: len(x), IntegerType()) lengthSchemaRDD = hiveCtx.sql("SELECT strLenPython('text') FROM tweets LIMIT 10")
```

例 9-37:Scala 版本的字符串长度 UDF

```scala
registerFunction("strLenScala", (_: String).length)
val tweetLength = hiveCtx.sql("SELECT strLenScala('tweet') FROM tweets LIMIT 10")
```

在 Java 中定义 UDF 需要一些额外的 import 声明。和在定义 RDD 函数时一样，根据我们 要实现的 UDF 的参数个数，需要扩展特定的类，如例 9-38 和例 9-39 所示。

例 9-38:Java UDF import 声明

```
// 导入UDF函数类以及数据类型
// 注意: 这些import路径可能会在将来的发行版中改变 

import org.apache.spark.sql.api.java.UDF1;
import org.apache.spark.sql.types.DataTypes;
```

例 9-39:Java 版本的字符串长度 UDF

```java
     hiveCtx.udf().register("stringLengthJava", new UDF1<String, Integer>() {
         @Override
           public Integer call(String str) throws Exception {
           return str.length();
         }
       }, DataTypes.IntegerType);
     SchemaRDD tweetLength = hiveCtx.sql(
       "SELECT stringLengthJava('text') FROM tweets LIMIT 10");
     List<Row> lengths = tweetLength.collect();
     for (Row row : result) {
       System.out.println(row.get(0));
     }
```

