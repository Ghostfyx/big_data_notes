# MapReduce Tutorial

## 1. 使用条件

确定Hadoop集群已经安装、配置并启动，伪分布和集群模式配置详情：

- [Single Node Setup](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html)
- [Cluster Setup](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/ClusterSetup.html) 

## 2. 概述

Hadoop MapReduce是一个软件框架，用于轻松编写分布式应用程序，应用程序以可靠，容错的方式并行处理大型硬件集群(数千个节点)上的大量数据(数TB数据集)。

MapReduce作业通常将输入数据集分成独立的块，这些任务由map任务以完全并行的方式处理。 计算框架对map任务输出排序，然后将其输入到reduce任务。通常作业的输入和输出都存储在文件系统中。 该框架负责安排任务，监视任务并重新执行失败的任务。

通常，计算节点和存储节点是相同的，即MapReduce框架和Hadoop分布式文件系统在同一组节点上运行。 此配置使框架可以在已经存在数据的节点上有效地调度任务，从而在整个群集中产生很高的聚合带宽。

MapReduce框架由一个master ResourceManager，集群每个节点上的nodeManager和每个应用的MRAppMaster构成。

应用程序通过适当的接口或抽象类来实现map、reduce方法并制定输入/输出路径。上述以及其他作业参数构成作业配置。

然后，Hadoop作业客户端将作业(jar/可执行文件等)和配置提交给ResourceManager，然后由ResourceManager负责将软件/配置分发给worker节点，调度任务并对其进行监视，向客户端提供应用的状态和运行诊断信息。

尽管Hadoop框架使用Java编写，但是MapReduce应用不必使用Java：

- Hadoop Streaming是一个实用程序，允许用户使用任何可执行文件(例如shell)，作为mapper和/或reducer来创建和运行作业。
- Hadoop Pipes是SWIG兼容的C ++ API，用于实现MapReduce应用程序(基于JNI™)

## 3. 输入与输出

MapReduce框架仅在<key，value>对上操作，即该框架将作业的输入视为一组<key，value>对，并生成一组<key，value>对作为任务的输出。map输入<K,V>的类型与reduce输出的<K,V>类型可能不相同。

键和值类必须由框架序列化，因此需要实现Writable接口。 另外，key的实体类必须实现WritableComparable接口，以方便框架进行排序。

MapReduce作业的输入和输出类型：

```
(input) <k1, v1> -> map -> <k2, v2> -> combine -> <k2, v2> -> reduce -> <k3, v3> (output)

```

## 4. WordCount v1.0示例

在进入细节之前，让我们来看一个MapReduce应用程序示例，以了解其工作方式。

WordCount是一个简单的应用程序，可以计算给定输入集中每个单词的出现次数。

### 4.1 代码

```java
import java.io.IOException;
import java.util.StringTokenizer;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class WordCount {

  public static class TokenizerMapper
       extends Mapper<Object, Text, Text, IntWritable>{

    private final static IntWritable one = new IntWritable(1);
    private Text word = new Text();

    public void map(Object key, Text value, Context context
                    ) throws IOException, InterruptedException {
      StringTokenizer itr = new StringTokenizer(value.toString());
      while (itr.hasMoreTokens()) {
        word.set(itr.nextToken());
        context.write(word, one);
      }
    }
  }

  public static class IntSumReducer
       extends Reducer<Text,IntWritable,Text,IntWritable> {
    private IntWritable result = new IntWritable();

    public void reduce(Text key, Iterable<IntWritable> values,
                       Context context
                       ) throws IOException, InterruptedException {
      int sum = 0;
      for (IntWritable val : values) {
        sum += val.get();
      }
      result.set(sum);
      context.write(key, result);
    }
  }

  public static void main(String[] args) throws Exception {
    Configuration conf = new Configuration();
    Job job = Job.getInstance(conf, "word count");
    job.setJarByClass(WordCount.class);
    job.setMapperClass(TokenizerMapper.class);
    job.setCombinerClass(IntSumReducer.class);
    job.setReducerClass(IntSumReducer.class);
    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);
    FileInputFormat.addInputPath(job, new Path(args[0]));
    FileOutputFormat.setOutputPath(job, new Path(args[1]));
    System.exit(job.waitForCompletion(true) ? 0 : 1);
  }
}
```

### 4.2 运行

配置环境变量

```sh
export JAVA_HOME=/usr/java/default
export PATH=${JAVA_HOME}/bin:${PATH}
export HADOOP_CLASSPATH=${JAVA_HOME}/lib/tools.jar
```

打包WordCount

```sh
bin/hadoop com.sun.tools.javac.Main WordCount.java
jar cf wc.jar WordCount*.class
```

输入/输出文件路径：

```sh
/user/joe/wordcount/input - input directory in HDFS
/user/joe/wordcount/output - output directory in HDFS

$ bin/hadoop fs -ls /user/joe/wordcount/input/
/user/joe/wordcount/input/file01
/user/joe/wordcount/input/file02

$ bin/hadoop fs -cat /user/joe/wordcount/input/file01
Hello World Bye World

$ bin/hadoop fs -cat /user/joe/wordcount/input/file02
Hello Hadoop Goodbye Hadoop
```

运行应用：

```sh
bin/hadoop jar wc.jar WordCount /user/joe/wordcount/input /user/joe/wordcount/output
```

输出结果：

```sh
hadoop fs -cat /user/joe/wordcount/output/part-r-00000

Bye 1
Goodbye 1
Hadoop 2
Hello 2
World 2
```

应用程序可以使用-files选项指定以逗号分隔的本地文件路径列表，这些路径将上传至任务的当前工作目录中。-libjars选项允许应用程序将jar添加到map、reduce类路径下。-archives选项将逗号分隔的归档列表作为参数传递，并且在当前任务工作目录中创建了带有归档文件名称的链接。 有关命令行选项的更多详细信息，请参见[Commands Guide](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/CommandsManual.html)。

使用-libjars，-files和-archives运行wordcount示例：

```sh
hadoop jar hadoop-mapreduce-examples-<ver>.jar wordcount -files cachefile.txt -libjars mylib.jar -archives myarchive.zip input output
```

在这里，myarchive.zip将被放置并解压缩到名为myarchive.zip的目录中。

用户可以使用＃为通过-files和-archives选项传递的文件和归档指定不同的别名。

```sh
bin/hadoop jar hadoop-mapreduce-examples-<ver>.jar wordcount -files dir1/dict.txt#dict1,dir2/dict.txt#dict2 -archives mytar.tgz#tgzdir input output
```

在这里，任务可以分别使用符号名称dict1和dict2访问文件dir1/dict.txt和dir2/dict.txt。 归档文件mytar.tgz将被放入名为tgzdir的目录下并取消归档。

应用程序可以通过在命令行上使用-Dmapreduce.map.env，-Dmapreduce.reduce.env和-Dyarn.app.mapreduce.am.env选项在命令行上分别指定map，reduce和appMaster的环境变量。

```sh
bin/hadoop jar hadoop-mapreduce-examples-<ver>.jar wordcount -Dmapreduce.map.env.FOO_VAR=bar -Dmapreduce.map.env.LIST_VAR=a,b,c -Dmapreduce.reduce.env.FOO_VAR=bar -Dmapreduce.reduce.env.LIST_VAR=a,b,c input output
```

