# Table API

Apache Flink为批处理和流处理提供统一的关系API——Table API，即查询在无边界的实时流或有约束的批处理数据集上以相同的语义执行，并产生相同的结果。 Flink中的Table API通常用于简化数据分析，数据管道和ETL应用程序的定义。

## 1. What Will You Be Building?

在本教程中，将学习如何建立一个连续的ETL管道，以随时间推移按帐户跟踪财务交易。 首先为报告构建为夜间批处理作业，然后迁移到流式处理管道。

```java
ExecutionEnvironment env   = ExecutionEnvironment.getExecutionEnvironment();
BatchTableEnvironment tEnv = BatchTableEnvironment.create(env);

tEnv.registerTableSource("transactions", new BoundedTransactionTableSource());
tEnv.registerTableSink("spend_report", new SpendReportTableSink());
tEnv.registerFunction("truncateDateToHour", new TruncateDateToHour());

tEnv
    .scan("transactions")
    .insertInto("spend_report");

env.execute("Spend Report");
```

## 2. Breaking Down The Code

### 2.1 The Execution Environment

前两行设置了ExecutionEnvironment。 执行环境可以为：Job设置属性，指定是编写批处理应用还是流应用程序以及创建source的方式。本章节示例从批处理环境开始，构建定期批处理报告。 然后将其包装在BatchTableEnvironment中，以完全访问Table API。

```java
ExecutionEnvironment env   = ExecutionEnvironment.getExecutionEnvironment();
BatchTableEnvironment tEnv = BatchTableEnvironment.create(env);
```

### 2.2 Registering Tables

接下来，在执行环境中注册表，可以使用这些表连接到外部系统以读取和写入批处理和流数据。Table Source提供对存储在外部系统中的数据的访问； 例如数据库，键值存储，消息队列或文件系统。Table Sink将表中数据发送到外部存储系统。根据source和sink的类型，支持不同的格式，例如CSV，JSON，Avro或Parquet。

```
tEnv.registerTableSource("transactions", new BoundedTransactionTableSource());
tEnv.registerTableSink("spend_report", new SpendReportTableSink());
```

注册两个表：交易输入表与支持报表输出表。通过交易表，可以读取信用卡交易，其中包含帐户ID(accountId)，时间戳(timestamp)和美元金额(accounts)。 在本教程中，输入表由内存中生成的数据支持，以避免对外部系统的任何依赖。 实际上，BoundedTransactionTableSource可以由文件系统，数据库或任何其他静态源支持。 支出报告(spend_report)表使用日志级别INFO记录每一行，而不是写入持久性存储，便于可以轻松查看结果。

### 2.3 Registering A UDF

与表一起，注册了用户定义的函数以使用时间戳。 此函数将时间戳四舍五入到最近的小时。

```java
tEnv.registerFunction("truncateDateToHour", new TruncateDateToHour());
```

### 2.4 The Query

配置了环境并注册了表之后，就可以构建第一个应用程序了。 在TableEnvironment中，可以扫描输入表以读取其行，然后使用insertInto将这些结果写入输出表中。

```java
tEnv
    .scan("transactions")
    .insertInto("spend_report");
```

### 2.5 Execute

Flink应用程序是延迟构建的，并仅在完全形成后才交付给集群以执行。 通过给ExecutionEnvironment.execute，以开始执行作业。

```
env.execute("Spend Report");
```

## 3. Attempt One

现在，有了Job设置的框架，您就可以添加一些业务逻辑。 目标是建立一个报告，以显示每个帐户在一天中每个小时的总支出。 就像SQL查询一样，Flink可以选择必填字段并按键进行分组。 由于时间戳字段具有毫秒级的粒度，因此可以使用UDF将其舍入到最接近的小时数。 最后，选择所有字段，并使用内置的汇总功能将每个帐户/小时对的总支出相加。

```java
tEnv
    .scan("transactions")
    .select("accountId, timestamp.truncateDateToHour as timestamp, amount")
    .groupBy("accountId, timestamp")
    .select("accountId, timestamp, amount.sum as total")
    .insertInto("spend_report");
```

该查询使用交易表中的所有记录，计算报告，并以有效，可扩展的方式输出结果。

```
# Query 1 output showing account id, timestamp, and amount

> 1, 2019-01-01 00:00:00.0, $567.87
> 2, 2019-01-01 00:00:00.0, $726.23
> 1, 2019-01-01 01:00:00.0, $686.87
> 2, 2019-01-01 01:00:00.0, $810.06
> 1, 2019-01-01 02:00:00.0, $859.35
> 2, 2019-01-01 02:00:00.0, $458.40
> 1, 2019-01-01 03:00:00.0, $330.85
> 2, 2019-01-01 03:00:00.0, $730.02
> 1, 2019-01-01 04:00:00.0, $585.16
> 2, 2019-01-01 04:00:00.0, $760.76
```

## 4. Adding Windows

基于时间对数据进行分组是数据处理中的典型操作，尤其是在处理无限流时。 基于时间的分组称为窗口，而Flink提供了灵活的窗口语义。 最基本的窗口类型称为滚动窗口，该窗口具有固定的大小，并且其存储桶不重叠。

```java
tEnv
    .scan("transactions")
    .window(Tumble.over("1.hour").on("timestamp").as("w"))
    .groupBy("accountId, w")
    .select("accountId, w.start as timestamp, amount.sum")
    .insertInto("spend_report");
```

这将应用程序定义为使用基于timestamp列的一小时滚动窗口。 因此，将带有时间戳2019-06-01 01:23:47的行放在2019-06-01 01:00:00窗口中。

基于时间的聚合是唯一的，因为与其他属性相反，时间通常在连续流应用程序中向前移动。 在批处理环境中，Windows提供了一种方便的API，用于按timestamp属性对记录进行分组。

运行更新的查询将产生与以前相同的结果。

```
# Query 2 output showing account id, timestamp, and amount

> 1, 2019-01-01 00:00:00.0, $567.87
> 2, 2019-01-01 00:00:00.0, $726.23
> 1, 2019-01-01 01:00:00.0, $686.87
> 2, 2019-01-01 01:00:00.0, $810.06
> 1, 2019-01-01 02:00:00.0, $859.35
> 2, 2019-01-01 02:00:00.0, $458.40
> 1, 2019-01-01 03:00:00.0, $330.85
> 2, 2019-01-01 03:00:00.0, $730.02
> 1, 2019-01-01 04:00:00.0, $585.16
> 2, 2019-01-01 04:00:00.0, $760.76
```

## 5. Once More, With Streaming

由于Flink的Table API为批处理和流提供了一致的语法和语义，因此从一个迁移到另一个只需两个步骤。

**第一步**：是将其批处理ExecutionEnvironment替换为其流媒体对应的StreamExecutionEnvironment，后者将创建一个连续的流作业。 它包括特定于流的配置，例如时间特性，将其设置为事件时间可确保即使遇到无序事件或作业失败也能保证一致的结果。 将记录分组时，“滚动”窗口将使用此功能。

```java
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

StreamTableEnvironment tEnv = StreamTableEnvironment.create(env);
```

**第二步**：是从有限数据源迁移到无限数据源。 该项目带有一个UnboundedTransactionTableSource，它可以连续不断地实时创建事务事件。 与BoundedTransactionTableSource相似，此表由内存中生成的数据支持，以避免对外部系统的任何依赖。 实际上，该表可以从诸如Apache Kafka，AWS Kinesis或Pravega之类的流源中读取。

```java
tEnv.registerTableSource("transactions", new UnboundedTransactionTableSource());
```

