# 第九章 MapReduce特性

本章探讨MapReduce的一些高级特性，其中包括计数器、数据集的排序和连接。

## 9.1 计数器

在许多情况下，用户需要了解待分析的数据，尽管这并非所要执行的分析任务的核心内容。以统计数据集中无效记录数目的任务为例，如果发现无效记录的比例 相当高，那么就需要认真思考为何存在如此多无效记录。是所采用的检测程序存在 缺陷，还是数据集质量确实很低，包含大量无效记录？如果确定是数据集的质量问 题，则可能需要扩大数据集的规模，以增大有效记录的比例，从而进行有意义的 分析。 

计数器是收集作业统计信息的有效手段之一，用于质量控制或应用级统计。计数器还可以辅助诊断系统故障。如果需要将日志信息传输到map或reduce任务，更好的方法通常是尝试传输计数器值以监测某一特定事件是否发生。对于大型分布式作业 而言，使用计数器更为方便。首先，获取计数器值比输出日志更方便，其次，根据 计数器值统计特定事件的发生次数要比分析一堆日志文件容易得多。

### 9.1.1 内置计数器

Hadoop为每个作业维护若干个内置计数器，以描述多项指标。例如：某些计数器记录已处理的字节数和记录数，使用户可监控以处理的输入数据量和已产生的输出数据量。

这些内置计数器被划分为若干组，参见表9-1。

​														**表9-1 内置计数器分组**

| 组别                   | 名称/类别                                                    | 参考  |
| ---------------------- | ------------------------------------------------------------ | ----- |
| MapReduce任务计数器    | org.apache.hadoop.mapreduce.TaskCounter                      | 表9-2 |
| 文件系统计数器         | org.apache.hadoop.mapreduce.FiIeSystemCounter                | 表9-3 |
| FiIeInputFormat计数器  | org.apache.hadoop.mapreduce.lib.input.FilelnputFormatCounter | 表9-4 |
| FiIeOutputFormat计数器 | org.apache.hadoop.mapreduce.lib.output.FileOutputFormatCounter | 表9-5 |
| 作业计数器             | org.apache.hadoop.mapreduce.JobCounter                       | 表9-6 |

各组要么包含任务计数器(在任务处理过程中不断更新)，要不包含作业计数器(在作业处理过程中不断更新)。这两种类型将在后续章节中进行介绍。

#### 1. 任务计数器

在执行任务过程中，任务计数器采集任务的相关信息，每个作业的所有任务的结果会被聚集起来。例如：MAP_INPUT_RECORDS计数器统计每个map任务输入记录的总数，并在一个作业的所有map任务上进行聚集，使得最终数字是这个作业的所有输入记录的总数。

任务计数器由其关联的任务维护，并定期发送给Application Master。因此，计数器能够被全局地聚集(参见7.1.5节中对进度和状态更新的介绍)。任务计数器的值每次都是完整传输的，而非传输自上次传输之后的计数值，从而避免由于消息丢失而引发的错误。另外，如果一个任务在作业执行期间失败，则相关计数器的值会减小。

虽然只有当这个作业执行完之后的计数器的值才是完整可靠的，但是部分计数器仍然可以在任务处理过程中提供一些有用的诊断信息，以便由Web页面监控。例如：PHYSICAL_MEMORY_BYTES、VIRTUAL_MEMORY_BYTES和COMMITTED_HEAP_BYTES计数器显示特定任务执行过程中的内存使用变化情况。

内置任务计数器包括在MapReduce任务计数器分组中的计数器(表9-2)以及在文件相关的计数器分组(表9-3，表9-4和表9-5)中的计数器。

​												**表 9-2 内置的MapReduce任务计数器**

| 计数器名称                                         | 说明                                                         |
| -------------------------------------------------- | ------------------------------------------------------------ |
| map输人的记录数(MAP_INPUT_RECORDS）                | 作业中所有map已处理的输人记录数。每次RecordReader读到一条记录并将其传给map的map()函数时，该计数器的值递增 |
| 分片（split）的原始字节数(SPLIT_RAW_BYTES)         | 由map读取的输人-分片对象的字节数。这些对象描述分片元数据（文件的位移和长度），而不是分片的数据自身，因此总规模是小的 |
| map输出的记录数(MAP_OUTPUT_RECORDS)                | 作业中所有map产生的map输出记录数。每次某一个map 的OutputCollector调用collect()方法时，该计数器的值增加 |
| map输出的字节数(MAP_OUTPUT_BYTES)                  | 作业中所有map产生的耒经压缩的输出数据的字节数·每次某一个map的OutputCollector调用collect()方法时，该计数器的值增加 |
| map输出的物化字节数(MAP_OUTPUT_MATERIALIZED_BYTES) | map输出后确实写到磁盘上的字节数；若map输出压缩功能被启用，则会在计数器值上反映出来 |
| combine输人的记录数(COMBINE_INPUT_RECORDS)         | 作业中所有combiner(如果有）已处理的输人记录数。combiner的迭代器每次读一个值，该计数器的值增加。注意：本计数器代表combiner已经处理的值的个数，并非不同的键组数（后者并无实所意文，因为对于combiner而言，并不要求每个键对应一个组，详情参见2.4.2节和7.3节 |
| combine输出的记录数(COMBINE_OUTPUT_RECORDS)        | 作业中所有combiner(如果有)已产生的输出记录数。每当一个combiner的OutputCollector调用collect()方法时，该计数器的值增加 |
| reduce输人的组(REDUCE_INPUT_GROUPS）               | 作业中所有reducer已经处理的不同的码分组的个数。每当某一个reducer的reduce()被调用时，该计数器的值增加 |
| reduce输人的记录数(REDUCE_INPUT_RECORDS)           | 作业中所有reducer已经处理的输人记录的个数。每当某个reducer的迭代器读一个值时，该计数器的值增加。如果所有reducer已经处理数完所有输人，則该计数器的值与计数器"map输出的记录"的值相同 |
| reduce输出的记录数(REDUCE_OUTPUT_RECORDS）         | 作业中所有map已经产生的reduce输出记录数。每当某个reducer的OutputCollector调用collect()方法时，该计数器的值增加 |
| reduce经过shuffle的字节数(REDUCE_SHUFFLE_BYTES)    | 由shume复制到reducer的map输出的字节数                        |
| 溢出的记录数(SPILLED_RECORDS)                      | 作业中所有map和reduce任务溢出到磁的记录数                    |
| CPU毫秒(CPU_MILLISECONDS)                          | 一个任务的总CPU时间，以毫秒为单位，可由/proc/cpuinfo获取     |
| 物理内存字节数(PHYSICAL_MEMORY_BYTES）             | 一个任务所用的物理内存，以字节数为单位，可 由/proc/meminfo获取 |
| 虚拟内存字节数(VIRTUAL_MEMORY_BYTES）              | 一个任务所用虚拟内存的字节数，由/proc/meminfo而'面获取       |
| 有效的堆字节数(COMMITTED_HEAP_BYTES)               | 在JVM中的总有效内存最（以字节为单位），可由Runtime. getRuntime().totalMemory()获取 |
| GC运行时间毫秒数(GC_TIME_MILLIS)                   | 在任务执行过程中，垃圾收集器(garbage collection）花费的时间（以毫秒为单位），可由GarbageCollector MXBean. getCollectionTime()获取 |
| 由shuffle传输的map输出数(SHUFFLED_MAPS)            | 由shuffle传输到reducer的map输出文件数，详情参见7.3节         |
| 失敗的shuffle数(FAILED_SHUFFLE)                    | shuffle过程中，发生map输出拷贝错误的次数                     |
| 被合并的map输出数(MERGED_MAP_OUTPUTS）             | shuffle过程中，在reduce端合并的map输出文件数                 |

​												**表9-3 内置文件系统任务计数器**

| 计数器名称                                 | 说明                                                         |
| ------------------------------------------ | ------------------------------------------------------------ |
| 文件系统的读字节数(BYTES_READ）            | 由map任务和reduce任务在各个文件系统中读取的字节数，各个文件系统分别对应一个计数器，文件系统可以是Local、 HDFS、S3等 |
| 文件系统的写字节数(BYTES_WRITTEN）         | 由map任务和reduce任务在各个文件系统中写的字节数              |
| 文件系统读操作的数量(READ_OPS)             | 由map任务和reduce任务在各个文件系统中进行的读操作的数量（例如，open操作，filestatus操作） |
| 文件系统大规模读操作的数最(LARGE_READ_OPS) | 由map和reduce任务在各个文件系统中进行的大规模读操作（例如，对于一个大容量目录进行list操作）的数 |
| 文件系统写操作的数最(WRITE_OPS)            | 由map任务和reduce任务在各个文件系统中进行的写操作的数量（例如，create操作，append操作） |

​													**表9-4 内置的FileInputFormat任务计数器**

| 计数器名称               | 说明                                     |
| ------------------------ | ---------------------------------------- |
| 读取的字节数(BYTES_READ) | 由map任务通过FilelnputFormat读取的字节数 |

​													**表9-5 内置的FileOutputFormat任务计数器**

| 计数器名称                | 说明                                                         |
| ------------------------- | ------------------------------------------------------------ |
| 写的字节数(BYTES_WRITTEN) | 由map任务（针对仅含map的作业）或者reduce任务通过FileOutputFormat写的字节数 |

#### 2. 作业计数器

作业计数器(表9-6)由Application Master维护，因此无需在网络间传输数据。这一点与包括“用户定义的计数器”在内的其他计数器不同。这些计数器都是作业级别的统计量，其值不会随着任务运行而改变。例如，TOTAL_LAUNCHED_MAPS统计在作业执行过程中启动的map任务数，包括失败的map任务。

​												**表9-6 内置的作业计数器**

| 计数器名称                                 | 说明                                                         |
| ------------------------------------------ | ------------------------------------------------------------ |
| 启用的map任务数(TOTAL_LAUNCHED_MAPS）      | 启动的map任务数，包括以“推测执行”方式启动的任务，详情参见7．4．2节 |
| 启用的reduce任务数(TOTAL_LAUNCHED_REDUCES) | 启动的reduce任务数，包括以“推测执行”方式启动的任务           |
| 启用的uber任务数(TOTAL_LAIÆHED_UBERTASKS)  | 启用的uber任务数，详情参见7.1节                              |
| uber任务中的map数(NUM_UBER_SUBMAPS)        | 在uber任务中的map数                                          |
| uber任务中的reduce数(NUM_UBER_SUBREDUCES)  | 在über任务中的reduce数                                       |
| 失败的map任务数(NUM_FAILED_MAPS）          | 失败的map任务数，用户可以参见7.2.1节对任务失败的讨论，了解失败原因 |
| 失败的reduce任务数(NUM_FAILED_REDUCES)     | 失败的reduce任务数                                           |
| 失败的uber任务数(NIN_FAILED_UBERTASKS)     | 失败的uber任务数                                             |
| 被中止的map任务数（NUM_KILLED_MAPS）       | 被中止的map任务数，可以参见7.2.1节对任务失败的讨论，了解中止原因 |
| 被中止的reduce任务数(NW_KILLED_REDUCES)    | 被中止的reduce任务数                                         |
| 数据本地化的map任务数(DATA_LOCAL_MAPS）    | 与输人数据在同一节点上的map任务数                            |
| 机架本地化的map任务数(RACK_LOCAL_MAPS)     | 与输人数据在同一机架范围内但不在同一节点上的map任务数        |
| 其他本地化的map任务数(OTHER_LOCAL_MAPS）   | 与输人数据不在同一机架范围内的map任务数。 由于机架之间的带宽资源相对较少，Hadoop会尽量让map任务靠近输人数据执行，因此该计数器值一般比较小。详情参见图2-2 |
| map任务的总运行时间(MILLIS_MAPS)           | map任务的总运行时间，单位毫秒。包括以推测执行方式启动的任务。可参见相关的度量内核和内存使用的计数器(VCORES_MILLIS_MAPS和MB_MILLIS_MAPS） |
| reduce任务的总运行时间(MILLIS_REDUCES)     | reduce任务的总运行时间，单位毫秒。包括以推滌执行方式启动的任务。可参见相关的度量内核和内存使用的计数器(VQES_MILLIS_REÄRES和t•B_MILLIS_REUKES) |

### 9.1.2 用户自定义计数器

MapReduce允许用户编写程序来定义计数器，计数器的值可在mapper或reducer中增加，计数器由一个Java枚举(enum)类型来定义，以便对有关的计数器分组。

一个作业可以定义的枚举类型数量不限，各个枚举类型所包含的字段数量也不限，枚举类型的名称即为组的名称，枚举类型的字段就是计数器名称。计数器是全局的，即MapReduce框架将跨所有map和reduce聚集这些计数器，并在作业结束时产生一个最终结果。

在第6章中，我们创建了若干计数器来统计天气数据集中不规范的记录数。范例9·1中的程序对此做了进一步扩展，能统计缺失记录和气温质量代码的分布情况。

**范例9．1，统计最高气温的作业，包括统计气温值缺失的记录、不规范的字段和质量代码**

```java
import java.io.IOException;

import org.apache.hadoop.conf.Configured;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.util.Tool;
import org.apache.hadoop.util.ToolRunner;

// vv MaxTemperatureWithCounters
public class MaxTemperatureWithCounters extends Configured implements Tool {
  
  enum Temperature {
    MISSING,
    MALFORMED
  }
  
  static class MaxTemperatureMapperWithCounters
    extends Mapper<LongWritable, Text, Text, IntWritable> {
    
    private NcdcRecordParser parser = new NcdcRecordParser();
  
    @Override
    protected void map(LongWritable key, Text value, Context context)
        throws IOException, InterruptedException {
      
      parser.parse(value);
      if (parser.isValidTemperature()) {
        int airTemperature = parser.getAirTemperature();
        context.write(new Text(parser.getYear()),
            new IntWritable(airTemperature));
      } else if (parser.isMalformedTemperature()) {
        System.err.println("Ignoring possibly corrupt input: " + value);
        context.getCounter(Temperature.MALFORMED).increment(1);
      } else if (parser.isMissingTemperature()) {
        context.getCounter(Temperature.MISSING).increment(1);
      }
      
      // dynamic counter
      context.getCounter("TemperatureQuality", parser.getQuality()).increment(1);
    }
  }
  
  @Override
  public int run(String[] args) throws Exception {
    Job job = JobBuilder.parseInputAndOutput(this, getConf(), args);
    if (job == null) {
      return -1;
    }
    
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);

    job.setMapperClass(MaxTemperatureMapperWithCounters.class);
    job.setCombinerClass(MaxTemperatureReducer.class);
    job.setReducerClass(MaxTemperatureReducer.class);

    return job.waitForCompletion(true) ? 0 : 1;
  }
  
  public static void main(String[] args) throws Exception {
    int exitCode = ToolRunner.run(new MaxTemperatureWithCounters(), args);
    System.exit(exitCode);
  }
}
```

将上述程序在完整的数据集上运行一遍：

```sh
hadoop jar hadoop-examples.jar MaxTemperatureWithCounters \
input/ncdc/all output-counters
```

作业成功执行完毕之后会输出各计数器的值(由作业的客户端完成)。以下是一组我们感兴趣的计数器的值。

```
Air Temperature Records
	Malformed = 3
	Missing=66136856
TemperatureQuality
	0=1
	1=973422173
	2=1246032
	4=10764500
	5=158291879
	6=40066
	9=66136858
```

注意，为使得气温计数器的名称可读性更好，使用了Java枚举类型的命名方式（使用下划线作为嵌套类的分隔符），这种方式称为“资源捆绑"，例如本例中为MaxTemperatureCounters_Temperature.properties，该捆绑包含显示名称的映射关系。

#### 1. 动态计数器

上述代码还使用了动态计数器，这是一种不由Java枚举类型定义的计数器。由于Java枚举类型的字段在编译阶段就必须指定，因而无法使用枚举类型动态新建计数器。范例8．1统计气温质量的分布，尽管通过格式规范定义了可以取的值，但相比之下，使用动态计数器来产生实际值更加方便。在该例中，context对象的`getcounter()`方法有两个string类型的输人参数，分别代表组名称和计数器名称：`public Counterget counter(String groupName，String counterName）`。

鉴于Hadoop需先将Java枚举类型转变成String类型，再通过RPC发送计数器值，这两种创建和访问计数器的方法（即使用枚举类型和String类型）事实上是等价的。相比之下，枚举类型易于使用，还提供类型安全，适合大多数作业使用。如果某些特定场合需要动态创建计数器，则可以使用String接口。

#### 2. 获取计数器

除了通过Web界面和命令行（执行mapred job -counter指令）之外，用户还可以使用JavaAPI获取计数器的值。通常情况下，用户一般在作业运行完成、计数器的值已经隐定下来时再获取计数器的值，而JavaAPI还支持在作业运行期间就能够获取计数器的值。范例9·2展示了如何统计整个数据集中气温信息缺失记录的比例。

**范例9.2 统计气温信息缺失记录所占的比例**

```java
mport org.apache.hadoop.conf.Configured;
import org.apache.hadoop.mapreduce.*;
import org.apache.hadoop.util.*;

public class MissingTemperatureFields extends Configured implements Tool {

  @Override
  public int run(String[] args) throws Exception {
    if (args.length != 1) {
      JobBuilder.printUsage(this, "<job ID>");
      return -1;
    }
    String jobID = args[0];
    Cluster cluster = new Cluster(getConf());
    Job job = cluster.getJob(JobID.forName(jobID));
    if (job == null) {
      System.err.printf("No job with ID %s found.\n", jobID);
      return -1;
    }
    if (!job.isComplete()) {
      System.err.printf("Job %s is not complete.\n", jobID);
      return -1;
    }

    Counters counters = job.getCounters();
    long missing = counters.findCounter(
        MaxTemperatureWithCounters.Temperature.MISSING).getValue();
    long total = counters.findCounter(TaskCounter.MAP_INPUT_RECORDS).getValue();

    System.out.printf("Records with missing temperature fields: %.2f%%\n",
        100.0 * missing / total);
    return 0;
  }
  public static void main(String[] args) throws Exception {
    int exitCode = ToolRunner.run(new MissingTemperatureFields(), args);
    System.exit(exitCode);
  }
}
```

首先，以作业ID为输入参数调用`getJob()`方法，从cluster中获取一个Job对象。通过检查返回是否为空来判断是否有一个作业与指定ID相匹配。有多种因素可能导致无法找到一个有效的Job对象，例如，错误地指定了作业ID，或是该作业不再保留在作业历史中。

其次，如果确认该作业已经完成，则调用该Job对象的getcounters()方法会返回一个counters对象，封装了该作业的所有计数器。counters类提供了多个方法用于获取计数器的名称和值。上例调用`findCounter()`方法，它通过一个枚举值来获取气温信息缺失的记录数和被处理的记录数(根据一个内置计数器)。

最后，输出气温信息缺失记录的比例。针对整个天气数据集的运行结果如下所示：

```
hadoop jar hadoop-examples.jar MissingTemperatureFields job_419425e506

Records with missing temperature fields：5.47％
```

## 9.2 排序

