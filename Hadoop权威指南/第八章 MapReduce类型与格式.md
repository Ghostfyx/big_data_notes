# 第八章 MapReduce类型与格式

MapReduce数据处理非常简单：map和reduce函数的输入和输出是键-值对。本章深入讨论MapReduce模型，重点介绍各类型的数据(从简单文本到结构化的二进制对象)如何在MapReduce中使用。

## 8.1 MapReduce类型

Hadoop的MapReduce中，map函数和reduce函数遵循如下常规格式：

```
map (K1, V1) --> list(K2, V2)
reduce(K2, list(V2)) --> list(K3, V3)
```

一般来说，map函数输入的键/值类型(K1,V1)不同于输出类型(K2, V2)。reduce函数的输入类型必须与map函数的输出类型相同，但reduce函数的输出类型(K3, V3)可以不同与输入类型。例如以下Java接口代码：

```java
public class Mapper<KEYIN, VALUEIN, KEYOUT, VALUEOUT> {
  public abstract class Context implements MapContext<KEYIN,VALUEIN,KEYOUT,VALUEOUT> {
  }
  
  protected void map(KEYIN key, VALUEIN value, Context context) throws IOException, InterruptedException {
    context.write((KEYOUT) key, (VALUEOUT) value);
  }
}

public class Reducer<KEYIN,VALUEIN,KEYOUT,VALUEOUT> {
  
  public abstract class Context implements ReduceContext<KEYIN,VALUEIN,KEYOUT,VALUEOUT> {
  }
  
  protected void reduce(KEYIN key, Iterable<VALUEIN> values, Context context) throws 	   IOException, InterruptedException {
    for(VALUEIN value: values) {
      context.write((KEYOUT) key, (VALUEOUT) value);
    }
  }
}
```

context对象用于输出键-值对，因此它们通过输出类型参数化，`wirte()`方法说明如下：

```java
public void write(KEYOUT key, VALUEOUT value) 
      throws IOException, InterruptedException;
```

由于Mapper和Reducer是单独的类，因此类型参数可能会不同，所以Mapper中KEYIN(say)的实际类型可能与Reducer中同名的类型参数（KEYIN）的类型不一致。例如，在前面章节的求最高温度例子中，Mapper中KEYIN为LongWritable类型，而Reducer中为Text类型。

如果使用combiner函数，它与reduce数（是Reducer的一个实现）的形式相同，不同之处是它的输出类型是中间的键·值对类型（K2和V2)，这些中间值可以输人reduce函数：

```
map:(K1,V1) -> list(K2,V2)
combiner:(K2 ,list(V2 ) ) -> list(K2 , V2)
reduce:(K2，list(V2)) -> list(K3,V3)
```

combiner函数与reduce函数通常是一样的，在这种情况下，K3与K2类型相同，V3与V2类型相同。

partition函数对中间结果的键·值对（K2和V2）进行处理，并且返回一个分区索引(partition index)。实际上，分区由键单独决定（值被忽略）。

```
partition：（K2,V2) -> integer
```

Java实现

```java
public abstract class Partitioner<KEY, VALUE> {
    public abstract int getPartition(KEY var1, VALUE var2, int var3);
}
```

表8-1总结了MapReduce API的配置选项，把属性分为可设置类型的属性和必须与类型相容的属性。

输入数据类型由输入格式进行设置，例如：对应于TextInputFormat的键类型是LongWritable，值类型是Text。其他的类型通过调用Job类的方法来进行显式设置(旧版本API中使用JobConf类的方法)。如果没有显式设置，则中间的类型默认为（最终的）输出类型，即默认值Longwritable和Text。因此，如果K2与K3是相同类型，就不需要调用`setMap0utputKeyClass()`，因为它将调用`set0utputKeyClass()`来设置；同样，如果V2与V3相同，只需要使用`setOutputValueClass()`。

这些为中间和最终输出类型进行设置的方法似乎有些奇怪。为什么不能结合mapper和reducer导出类型呢？ 因为，Java的泛类型机制有很多限制：类型擦除(type erasure)导致运行过程中类型信息并非一直可见，所以Hadoop不得不进行明确设定。这也意味着可能会在MapReduce配置的作用中遇到不兼容的类型，因为这些配置在编译时无法检查。与MapReduce类型兼容的设置列在表8-1中。类型冲突是在作业执行过程中被检测出来的，所以一个比较明智的做法是先用少量数据跑一次测试任务，发现并修正任何一个类型不兼容的问题。

​														**表8-1 MapReduce API中的设置类型**

| 属性                                        | 属性设置方法                 | 输入类型K1 | V1   | 中间类型K2 | V2   | 输出类型K3 | V3   |
| ------------------------------------------- | ---------------------------- | ---------- | ---- | ---------- | ---- | ---------- | ---- |
| 可以设置类型的属性                          |                              |            |      |            |      |            |      |
| mapreduce.job.inputformat.class             | setInputFormatClass()        |            | *    |            |      |            |      |
| mapreduce.map.output.key.class              | setMapOutputKeyClass()       |            |      | *          |      |            |      |
| mapreduce.map.output.value.class            | setMapOutputValueClass()     |            |      |            | *    |            |      |
| mapreduce.job.output.key.class              | setOutputKeyClass()          |            |      |            |      | *          |      |
| mapreduce.job.output.value.class            | setOutputValueClass()        |            |      |            |      |            | *    |
| 类型必须一致的属性                          |                              |            |      |            |      |            |      |
| mapreduce.job.map.class                     | setMapperClass()             | *          | *    | *          | *    |            |      |
| mapreduce.job.combine.class                 | setCombinerClass()           |            |      | *          | *    |            |      |
| mapreduce.job.partitioner.class             | setPartitionerClass()        |            |      | *          | *    |            |      |
| mapreduce.job.output.key.comparator.class   | setSortComparatorClass()     |            |      |            |      |            |      |
| mapreduce.job.output.group.comparator.class | setGroupingComparatorClass() |            |      | *          |      |            |      |
| mapreduce.job.reduce.class                  | setReducerClass()            |            |      | *          | *    | *          | *    |
| mapreduce.job.outputformat.class            | setOutputFormatClass()       |            |      |            |      | *          | *    |

### 8.1.1 默认的MapReduce作业

如果不指定mapper或reducer就运行MapReduce，会发生什么情况？运行一个最简单的MapReduce程序来看看：

```java
public class MinimalMapReduce extends Configured implements Tool {
 
    @Override
    public int run(String[] args) throws Exception {
        if(args.length!=2){
            System.err.printf("Usage: %s [generic options] <input> <output>\n",getClass().getSimpleName());
            ToolRunner.printGenericCommandUsage(System.err);
        }
        Job job=Job.getInstance(getConf());
        job.setJarByClass(getClass());
        FileInputFormat.addInputPath(conf,new Path(args[0]));
        FileOutputFormat.setOutputPath(conf,new Path(args[1]));
        return job.waitForCompletion(true)?0:1;
    }
 
    public static void main(String[] args) throws Exception {
        int exitCode=ToolRunner.run(new MinimalMapReduce(),args);
        System.exit(exitCode);
    }
}
```

唯一设置的是输入输出路径。在气象数据的子集上运行一下命令：

```sh
hadoop MinimalMapReduce  "input/ncdc/all/190{1,2}.gz" output
```

输出目录中得到命名为part-r-00000的输出文件。这个文件的前几行如下(为适应页面而进行了截断处理）:

![](./img/8-part.png)

每一行以整数开始，接着是制表符(Tab)，然后是一段原始气象数据记录。范例8．1的示例与前面MinimalMapReduce完成的事情一模一样，但是它显式地把作业环境设置为默认值。

```java
public class MinimalMapReduceWidthDefaults extends Configured implements Tool {
 
    @Override
    public int run(String[] args) throws Exception {
        if(args.length!=2){
            System.err.printf("Usage: %s [generic options] <input> <output>\n",getClass().getSimpleName());
            ToolRunner.printGenericCommandUsage(System.err);
        }
        Job job=Job.getInstance(getConf());
        job.setJarByClass(getClass());
        job.setInputFormatClass(TextInputFormat.class)
        job.setMapperClass(Mapper.class);
        job.setMapOutputKeyClass(LongWritable.class);
        job.setMapOutputValueClass(Text.class);
        job.setPartitionerClass(HashPartitioner.class);
        job.setNumReduceTasks(1);
        job.setReducerClass(Reducer.class);
        job.setOutputKeyClass(LongWritable.class);
        job.setOutputValueClass(Text.class);
        return job.waitForCompletion(true)?0:1;
    }
 
    public static void main(String[] args) throws Exception {
        int exitCode=ToolRunner.run(new MinimalMapReduce(),args);
        System.exit(exitCode);
    }
}
```

在默认的输人格式是TextInputFormat，它产生的键类型是LongWritable(文件中每行中开始的偏移量值)，值类型是Text(文本行)。

默认的mapper是Mapper类，它将输人的键和值原封不动地写到输出中：

```java
public class Mapper<KEYIN, VALUEIN, KEYOUT, VALUEOUT> {
    protected void map(KEYIN key, VALUEIN value, Mapper<KEYIN, VALUEIN, KEYOUT, VALUEOUT>.Context context) throws IOException, InterruptedException {
        context.write(key, value);
    }
}
```

Mapper是一个泛型类型(generictype)，它可以接受任何键或值的类型。在这个例子中，map的输人输出键是LongWritable类型，map的输人输出值是Text类型。

默认的partitioner是HashPartitioner，它对每条记录的键进行哈希操作以决定该记录应该属于哪个分区。每个分区由一个reduce任务处理，所以分区数等于作业的reduce任务个数：

```java
public class HashPartitioner<K, V> extends Partitioner<K, V> {
    public int getPartition(K key, V value, int numReduceTasks) {
        return (key.hashCode() & 2147483647) % numReduceTasks;
    }
}
```

键的哈希码被转换为一个非负整数，它由哈希值与最大的整型值做一次按位与操作而获得，然后用分区数进行取模操作，来决定该记录属于哪个分区索引。

默认情况下，只有一个reducer，因此，也就只有一个分区，在这种情况下，由于所有数据都放人同一个分区，partitioner操作将变得无关紧要了。然而，如果有多个reduce任务，了解HashPartitioner的作用就非常重要。假设基于键的散列函数足够好，那么记录将被均匀分到若干个reduce任务中，这样，具有相同键的记录将由同一个reduce任务进行处理。

map任务的数量等于输人文件被划分成的分块数，这取决于输人文件的大小以及文件块的大小（如果此文件在HDFS中）。关于控制块大小的操作，可以参见8.2.1节。

**选择Reducer个数**

对Hadoop新手而言，单个reducer的默认配置很容易上手。但在真实的应用中，几乎所有作业都把它设置成一个较大的数字，否则由于所有的中间数据都会放到一个reduce任务中，作业处理极其低效。

为一个作业选择多少个reducer与其说是一门技术，不如说更多是一门艺术。由于并行化程度提高，增加reducer数量能缩短reduce过程，然而，如果做过了，小文件将会更多，这又不够优化。一条经验法则是：**目标reducer保持在每个运行5分钟左右、且产生至少一个HDFS块的输出比较合适。**

默认的reducer是Reducer类型，它也是一个泛型类型，只是把所有的输人写到输出中：

```java
public class Reducer<KEYIN, VALUEIN, KEYOUT, VALUEOUT> {
       protected void reduce(KEYIN key, Iterable<VALUEIN> values, Reducer<KEYIN, VALUEIN, KEYOUT, VALUEOUT>.Context context) throws IOException, InterruptedException {
        Iterator var4 = values.iterator();
        while(var4.hasNext()) {
            VALUEIN value = var4.next();
            context.write(key, value);
        }
    }
}
```

对于这个任务来说，输出的键是LongWritable类型，而值是Text类型。事实上，对于这个MapReduce程序来说，所有键都是LongWritable类型，所有值都是Text类型，因为它们是输人键/值，并且map函数和reduce函数是恒等函数。然而，大多数MapReduce程序不会一直用相同的键或值类型，必须配置作业来声明使用的类型。

mapper结果在发送给reducer之前，会被MapReduce系统按照键排序。在这个例子中，键是按照数值的大小进行排序的，因此来自输人文件中的行会被交叉放人一个合并后的输出文件(因为默认只有一个reducer)。

默认的输出格式是Text0utputFormat，它将键和值转换成字符串并用制表符分隔开，然后一条记录一行地进行输出。这是TextOutputFormat的特点。