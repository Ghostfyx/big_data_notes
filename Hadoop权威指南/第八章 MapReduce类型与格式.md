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

## 8.2 输入格式

从一般的文本文件到数据库，Hadoop可以吹很多不同类型的数据格式。

### 8.2.1 输入分片与记录

一个输入分片(Split)就是一个由单个map操作来处理的输入块。每一个map操作只处理一个输入分片，每个分片被划分为若干个记录，每条记录就是一个键-值对，map一个接一个处理。输入分片和记录都是逻辑概念，不必将它们对应到文件，尽管其常见形式都是文件。在数据库场景中，一个输入分片可以对应于一个表上的若干行，而一条记录对应到一行(如同DBInputFormat，这种输入格式用于从关系型数据库读取数据)。

输入分片在Java中表示为InputSplit接口（在org.apache.hadoop.mapreduce包中）。

```java
public abstract class InputSplit {
	public abstract long getLength() throws IOException, InterruptedException;
	public abstract String[] getLocations() throws IOException, InterruptedException;
}
```

InputSplit包含一个以字节为单位的长度和一组存储位置(即一组主机名)。注意，分片并不包含数据本身，而是指向数据的引用(reference)。存储位置供MapReduce系统使用以便将map任务尽量放在分片数据附近，而分片大小用来排序分片，以便优先处理最大的分片，从而最小化作业运行时间(这也是贪婪近似算法的一个实例)。

MapReduce应用开发人员不必直接处理InputSplit，因为它是由InputFormat创建的(InputFormat负责创建输入分片并将它们分隔成记录)，它在MapReduce中的用法。接口如下：

```java
public abstract class InputFormat<K,V> {
	public abstract List<InputSplit> getSplits(JobContext context)
		throws  IOException, InterruptedException;

	public abstract RecordReader<K,V> createRecordReader(InputSplit split, TaskAttemptContext context)
		throws IOException, InterruptedException;
}
```

运行作业的客户端通过调用`getSplit()`计算分片，然后将它门发送到application master，application master使用其存储位置信息调度map任务从而在集群上处理分片数据。map任务把输入分片传给`InputFormat`的`createRecordReader()`方法来获取分片的`RecordReader`。`RecordReader`就像是记录上的迭代器，map任务用一个RecordReader来生成记录的键-值对，然后再传递给map函数。Mapper的run()方法可以看到这些情况：

```java
protected void map(KEYIN key, VALUEIN value, 
                     Context context) throws IOException, InterruptedException {
    context.write((KEYOUT) key, (VALUEOUT) value);
}

public void run(Context context) throws IOException, InterruptedException {
	setup(context);
	while (context.netKeyValue()) {
		map(context.getCurrentKey(), context.getCurrentValue(), context);
	}
	cleanup(context);
}
```

运行setup()之后，再重复调用Context上的nextKeyValue()(委托给RecordReader的同名方法)为mapper产生键-值对象。通过Context，键/值从RecordReader中被检索出并传递给map()方法。当reader读到stream的结尾时，nextKeyValue()方法返回false，map任务运行其cleanup方法，然后结束。

```java
public abstract class RecordReader<KEYIN, VALUEIN> implements Closeable {
  
  /**
   * Read the next key, value pair.
   * @return true if a key/value pair was read
   * @throws IOException
   * @throws InterruptedException
   */
  public abstract 
  boolean nextKeyValue() throws IOException, InterruptedException;

  /**
   * Get the current key
   * @return the current key or null if there is no current key
   * @throws IOException
   * @throws InterruptedException
   */
  public abstract
  KEYIN getCurrentKey() throws IOException, InterruptedException;
  
  /**
   * Get the current value.
   * @return the object that was read
   * @throws IOException
   * @throws InterruptedException
   */
  public abstract 
  VALUEIN getCurrentValue() throws IOException, InterruptedException;
}
```

注意Mapper的run()方法是公共的，可以由用户定制。MultithreadedMapRunner是另一个MapRunnable接口的实现，可以用可配置个数的线程并发运行多个mapper(mapreduce.mapper.multithreadedmapper.threads设置）。对于大多数数据处理任务来说，默认的执行机制没有优势。但是，对于因为需要连接外部服务器而造成单个记录处理时间比较长的mapper来说，它允许多个mapper在同一个JVM下以尽量避免竞争的方式执行。

#### 1. FileInputFormat类

FileInputFormat是所有使用文件作为其数据源的InputFormat实现的基类，它提供两个功能：

- 用于指出作业的输入文件位置；
- 为输入文件生成分片的代码实现。把分片分割成记录的作业由其子类来完成。

![](./img/8-2.jpg)

​													**图 8.2 InputFormat 类的层次结构**

#### 2. FileInputFormat类的输入路径

作业的输入被设定为一组路径，这对限定输入提供了很强的灵活性。FileInputFormat提供四种静态方法来设定Job的输入路径：

```java
public static void addInputPath(Job job, Path path)
public static void addInputPaths(Job job, String commaSeparatedPaths)
public static void setInputPaths(Job job, Path... inputPaths)
public static void setInputPaths(Job job, String commaSeparatedPaths)
```

介绍：

- addInputPath 添加一个路径
- addInputPaths 添加多个路径
- setInputPaths 一次性设定完完整的路径(替换前面调用Job中所设置的路径)

一条路径可以表示一个文件，一个目录或是一个glob，即一个文件和目录的集合。路径是目录的话，表示要包含这个目录下的所有文件。

**注意：**一个被指定为输入路径的目录，其内容不会被递归处理。如果包含子目录，也会被解释为文件，从而产生错误。解决这个问题的方法是：

- 使用一个文件glob或一个过滤器根据命名模式(name pattern)限定选择目录中的文件。
- 将`mapreduce.input.fileinputformat.input.dir.recursive`设置为true，强制递归读取

add方法和set方法允许指定包含的文件。排除特定文件，可用`FileInputFormat`的`setInputPathFilter()`方法设置一个过滤器。过滤器的详细讨论参见3.5.5节中对PathFilter的讨论。

即使不设置过滤器，FileInputFormat也会使用一个默认的过滤器来排除隐藏文件（名称中以“.”和“_”开头的文件）。如果通过调用setInputPathFilter()设置了过滤器，它会在默认过滤器的基础上进行过滤。换句话说，自定义的过滤器只能看到非隐藏文件。

路径和过滤器也可以通过配置属性来设置，参见表8-4：

​													**表8-4 输入路径与过滤器属性**

| 属性名称                                 | 类型           | 默认值 | 描述                                                         |
| ---------------------------------------- | -------------- | ------ | ------------------------------------------------------------ |
| mapreduce.input.fileinputformat.inputdir | 逗号分隔的路径 | 无     | 作业的输人文件。包含逗号的路径中的逗号由“\”符号转义。例如，glob{a,b}变成了{a\,b} |
| mapreduce.input.pathFilter.class         | PathFilter类名 | 无     | 应用于作业输人文件的过滤器                                   |

#### 3. FileInputFormat类的输入分片

假设有一组文件，FileInputFormat如何把它们转换为输人分片呢？`FileInputFormat`只分割大文件。这里的“大”指的是文件超过HDFS块的大小。分片通常与HDFS块大小一样，这在大多应用中是合理的。这个值可以通过设置不同的Hadoop属性来改变，如表8-5所示：

​														**表 8-5 控制分片大小的属性**

| 属性名称                                      | 类型 | 默认值             | 描述                                       |
| --------------------------------------------- | ---- | ------------------ | ------------------------------------------ |
| mapreduce.input.fileinputformat.split.minsize | int  | 1                  | 一个文件分片最小的有效字节数               |
| mapreduce.input.fileinputformat.split.maxsize | long | Long.MAX_VALUE     | 一个文件分片中最大的有效字节数（以字节算） |
| dfs.blocksize                                 | long | 128MB，即134217728 | HDFS中块的大小（按字节）                   |

最大的分片大小默认是由Java的long类型表示的最大值。只有把它的值被设置成小于块大小才有效果，这将强制分片比块小。

分片大小由以下公式计算，参见FileInputFormat的computeSplitSize()方法：

```java
max(minimumSize，min(maximumSize,blockSize））
```

在默认情况下：

```
minimumSize < blockSize < maximumSize
```

分片的大小就是blocksize，参数的不同设置及其如何影响最终分片大小，请参见表8·6的详细说明。

​													**表8-6 举例说明如何控制分片的大小**

| 最小份片大小 | 最大分片大小             | 块的大小        | 分片大小 | 说明                                                         |
| ------------ | ------------------------ | --------------- | -------- | ------------------------------------------------------------ |
| 1(默认值)    | Long.MAX_VALUE（默认值） | 128MB（默认值） | 128MB    | 默认情况下，分片大小与块大小相同                             |
| 1（默认值）  | Long.MAX_VALUE（默认值） | 256MB           | 256MB    | 增加分片大小最自然的方法是提供更大的HDFS 块，通过dfs.blocksize或在构建文件时以单个文件为基础进行设置 |
| 256MB        | Long.MAX_VALUE（默认值） | 128MB（默认值） | 256MB    | 通过使最小分片大小的值大于块大小的方法来增大 分片大小，但代价是增加了本地操作 |
| 1（默认值）  | 64MB                     | 128MB（默认值） | 64MB     | 通过使最大分片大小的值小于块大小的方法来减少分片大小         |

#### 4. 小文件与CombineFileInputFotmat

相对于处理大批量的小文件，Hadoop更适合处理少量的大文件。一个原因是FileInputFormat生成的块是一个文件或文件的一部分，如果文件很小（“小”意味着比HDFS的块要小很多），并且文件数量很多，那么每次map任务只处理很少的输人数据，一个文件就会有很多map任务，每次map操作都会造成额外的开销。请比较一下把1GB的文件分割成8个128MB块与分成1000个左右100KB的文件。1000个文件每个都需要使用一个map任务，作业时间比一个输人文件上用8个map任务慢几十倍甚至几百倍。

CombineFileInputFormat可以缓解这个问题，它是针对小文件而设计的。FileInputFomnat为每个文件产生1个分片，而CmbineFileInputFomat把多个文件打包到一个分片中以便每个mapper可以处理更多的数据。关键是，决定哪些块放人同一个分片时，CombineFileInputFormat会考虑到节点和机架的因素，所以在典型MapReduce作业中处理输人的速度并不会下降。

当然，如果可能的话应该尽量避免许多小文件的情况，因为：

- MapReduce处理数据的最佳速度最好与数据在集群中的传输速度相同，而处理小文件将增加运行作业而必需的寻址次数。
- HDFS集群中存储大量的小文件会浪费namenode的内存。

一个可以减少大量小文件的方法是使用顺序文件（sequence file）将这些小文件合并成一个或多个大文件，可以将文件名作为键（如果不需要键，可用NullWritable等常量代替），文件的内容作为值。但如果HDFS中已经有大批小文件，可以使用CombineFileInputFormat方法。

CombineFileInputFormat不仅可以很好地处理小文件，在处理大文件的时候也有好处。这是因为，它在每个节点生成一个分片，分片可能由多个块组成。本质上，combineFileInputFormat使map操作中处理的数据量与HDFS中文件的块大小之间的耦合度降低了。

#### 5.避免切分

有些应用程序可能不希望文件被切分，而是用一个mapper完整处理每一个输人文件。例如，检查一个文件中所有记录是否有序，一个简单的方法是顺序扫描每一条记录并且比较后一条记录是否比前一条要小。如果将它实现为一个map任务，那么只有一个map操作整个文件时，这个算法才可行（SortValidator.RecordStatsChecker中的mapper就是这样实现的）。

有两种方法可以保证输入文件不被切分：

-  增加最小分片的大小，将它设置成要处理的最大文件大小，设置为最大值Long.MAX_VALUE即可；

- 使用FileInputFormat具体子类，并且重写isSplitable()方法。把返回值设置为false。例如，以下就是一个不可分割的TextInputFormat：

	```java
	public class NonSplittableTextInputFormat extends TextInputFormat{
	      @override
	       protected boolean isSplitable(JobContext context,Path file){
	                return false;
	       }
	}
	```

#### 6. mapper中的文件信息

处理文件输入分片的mapper可以从作业配置对象的某些特定属性中读取分片的有关信息，可以通过调用在Mapper的Context对象上的getInputSplit()方法来实现。当输人的格式源自于FileInputFormat时，该方法返回的InputSplit可以被强制转换为一个FileSplit，以此来访问表8-7列出的文件信息。

​																**表8-7 文件输入分片的属性**

| FileSplit方法 | 属性名称                 | 类型        | 说明                     |
| ------------- | ------------------------ | ----------- | ------------------------ |
| getPath()     | mapreduce.map.input.file | Path/String | 正在处理的输人文件的路径 |
| getStart()    | mapreducenp.input.start  | long        | 分片开始处的字节偏移量   |
| getLength()   | mapreducenp.input.length | long        | 分片的长度（按字节）     |

#### 7. 把整个文件作为一条记录处理

有时，mapper需要访问一个文件中的全部内容。即使不分割文件，仍然需要一个RecordReader来读取文件内容作为record的值。范例的WhoIeFiIeInputFormat展示了实现的方法。

**例8-2 把整个文件作为一条记录的InputFotmat**

```java
public class WholeFileInputFormat extends FileInputFormat<NullWritable,ByteWritable>{
        @Override
        protected boolean isSplitable(JobContext context, Path filename) {
            return false;
        }
        @Override
        public RecordReader<NullWritable, ByteWritable> createRecordReader(InputSplit inputSplit, TaskAttemptContext taskAttemptContext) throws IOException, InterruptedException {
            WholeFileRecordReader reader = new WholeFileRecordReader();
            reader.initalize(inputSplit,taskAttemptContext);
            return reader;
        }
    }
```

`WholeFileInputFormat`中没有使用键，此处表示为`NullWritable`，值是文件内容，表示成`BytesWritable`实例。它定义了两个方法：一个是将`isSplitable()`方法重写返回false值，以此来指定输人文件不被分片；另一个是实现了`createRecordReader()`方法，以此来返回一个定制的`RecordReader`实现，如范例8．3所示。

**范例8- 3 WholeFilelnputFormat使用RecordReader将整个文件读为一条记录**

```java
class WholeFileRecordReader extends RecordReader<NullWritable,BytesWritable>{
  			private FileSplit fileSplit;
        private Configuration conf;
        private BytesWritable value=new BytesWritable();
        private boolean processed=false;
  			
  			@Override
        public void initialize(InputSplit inputSplit, TaskAttemptContext taskAttemptContext) throws IOException, InterruptedException {
            this.fileSplit=(FileSplit) inputSplit;
            this.conf=taskAttemptContext.getConfiguration();
        }
  			
  			@Override
        public boolean nextKeyValue() throws IOException, InterruptedException {
            if(!processed){
                byte[] contents=new byte[(int)fileSplit.getLength()];
                Path file=fileSplit.getPath();
                FileSystem fs=file.getFileSystem(conf);
                FSDataInputStream in=null;
                try{
                    in=fs.open(file);
                    IOUtils.readFully(in,contents,0,contents.length);
                    value.set(contents,0,contents.length);
                }finally{
                    IOUtils.closeStream(in);
                }
                processed=true;
                return true;
            }
            return false;
        }
  			
  			@Override
        public NullWritable getCurrentKey() throws IOException, InterruptedException {
            return NullWritable.get();;
        }
 
        @Override
        public BytesWritable getCurrentValue() throws IOException, InterruptedException {
            return value;
        }
 
        @Override
        public float getProgress() throws IOException, InterruptedException {
            return processed?1.0f:0.0f;
        }
 
        @Override
        public void close() throws IOException {
            //do nothing
        }

}
```

WholeFileRecordReader负责将FileSplit转换成一条记录，该记录的键是null，值是这个文件的内容。因为只有一条记录，WholeFileRecordReader要么处理这条记录，要么不处理，所以它维护一个名称为processed的布尔变量来表示记录是否被处理过。如果当nextKeyvalue()方法被调用时，文件没有被处理过，就打开文件，产生一个长度是文件长度的字节数组，并用Hadoop的IOUtils类把文件的内容放人字节数组。然后再被传递到next()方法的BytesWritable实例上设置数组，返回值为true则表示成功读取记录。

其他一些方法都是一些直接的用来访问当前的键和值类型、获取reader进度的方法，还有一个close()方法，该方法由MapReduce框架在reader完成后调用。

现在演示使用WholeFileInputFormat。假设有一个将若干个小文件打包成顺序文件的MapReduce作业，键是原来的文件名，值是文件的内容。如例8-4所示

```java
public class SmallFilesToSequenceFileConverter extends Configured implements Tool{
        public static void main(String[] args) {
            int exitCode=ToolRunner.run(new SmallFilesToSequenceFileConverter(),args);
            System.exit(exitCode);
        }
        
        @Override
        public int run(String[] args) throws Exception {
            Job job=JobBuilder.parseInputAndOutput(this,getConf(),args);
            if(conf==null){
                return -1;
            }
            job.setInputFormatClass(WholeFileInputFormat.class);
            job.setOutputFormatClass(SequenceFileOutputFormat.class);
            job.setOutputKeyClass(Text.class);
            job.setOutputValueClass(BytesWritable.class);
            job.setMapperClass(SequenceFileMapper.class);
            return job.waitForCompletion(true)?0:1;
        }
 
        static class SequenceFileMapper extends Mapper<NullWritable,BytesWritable,Text,BytesWritable>{
            private Text filenameKey;
            @Override
            protected void setup(Context context) throws IOException, InterruptedException {
                InputSplit split=context.getInputSplit();
                Path path=((FileSplit)split).getPath();
                filenameKey=new Text(path.toString());
            }
 
            @Override
            protected void map(NullWritable key, BytesWritable value, Context context) throws IOException, InterruptedException {
                context.write(filenameKey,value);
            }
        }
    }
```

