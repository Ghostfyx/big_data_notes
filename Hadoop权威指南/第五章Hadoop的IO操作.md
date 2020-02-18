## 第五章 Hadoop的I/O操作

Hadoop自带一套原子操作用于数据I/O操作，其中有一些技术比Hadoop本身更加有用，如数据完整性和数据压缩，其他一些则是Hadoop工具或API，它们形成的模块可用于开发分布式文件系统。

## 5.1 数据完整性

检测数据是否损坏的常用方式是，在数据第一次引入系统时**计算校验和**(checkSum)并在数据通过一个不可靠通道进行传输时再次计算校验和，如果校验和不一致，则认为数据已经损坏。但该技术不能修复数据，因此应该避免使用低端硬件，具体来说，一定要使用ECC内存(应用了能够实现错误检查和纠正技术的内存条。一般多应用在服务器及图形工作站上，这将使系统在工作时更趋于安全稳定)。

常用的错误检测码是CRC-32(32位循环冗余校验)，任何大小的数据输入均计算得到一个32位的整数校验和。Hadoop的ChecksunFileSystem使用CRC-32计算校验和，HDFS使用变体CRC-32计算，

### 5.1.1 HDFS的数据完整性

HDFS会对所有写入的数据计算校验和，并在读取数据时验证校验和。HDFS针对每个由dfs.bytes-per-checksum指定字节数的数据计算校验和，默认512字节，由于CRC-32校验和是4个字节，所以校验和的额外开销低于1%。

datanode负责在收到数据后**存储该数据及其校验和**之前对数据进行验证。它在收到客户端数据或复制到其他datanode的数据时执行这个操作。正在写数据的客户端将数据及其校验和发送到一系列datanode组成的管线，管线中的最后一个datanode负责验证校验和，如果datanode检测到错误，客户端接收到一个IOException的子类，对于该异常应以应用程序特定的方式处理，比如重试这个操作。

**客户端从datanode读取数据**时，会验证校验和，将它们与datanode存储的校验和比较。每个datanode持久化保存有一个用于验证校验和日志(persistent log of checksum verification) 。客户端成功验证一个数据块后，datanode更新校验和日志，保存这些统计信息对与检测损坏的磁盘很有价值。

datanode自身运行后台线程DataBlockScanner，定期验证存储在datanode上所有数据块。该项措施是解决物理存储媒体上位损坏的有力措施。

由于HDFS存储着每个数据块的复本(replica)，因此可以通过数据复本来修复损坏数据。基本思路如下：客户端在读取数据时，验证校验和错误：

- 客户端向namenode报告已损坏数据及其正在尝试读操作的datanode，抛出ChecksumException。
- namenode将这个数据块复本标记为已损坏，不再将客户端处理请求直接发送到节点。
- Namenode安排这个数据块的一个复本复制到另一个datanode，如此一来，数据块的复本因子(replication factor)又回到期望水平，删除已损坏的数据块复本。

在使用open( )方法读取文件之前，将FileSystem对象的setVerifyChecksum( )方法设置为false，即可以禁用校验和验证。在命令解释器中使用带-get选项的-ignoreCrc命令或者使用等价的-copyToLocal命令，也可以达到相同的效果。

可以用hadoop 的命令fs –checksum来检查一个文件的校验和。这可用于在HDFS中检查两个文件是否具有相同内容，distcp命令也具有类似的功能。详情可以参见3.7节。

### 5.1.2 LocalFileSystem

Hadoop的LocalFileSystem执行客户端的校验和验证。在写入一个名为filename的文件时，文件系统客户端会明确在包含每个文件块校验和的同一目录新建一个filename.crc隐藏文件。文件块大小作为元数据存储在.crc文件中，即使数据块的大小的设置发生变化，仍然可以正确读回文件(理解：系统数据块大小设置发生改变，对于历史数据的兼容)。在**读取文件**时需要验证校验和，如果检测到错误，LocalFileSystem会抛出ChecksumException异常。

校验和的计算代价是相当低的(在Java中，它们是用本地代码实现的)，一般只是增加少许额外的读/写文件时间。对大多数应用来说，付出这样的额外开销以保证数据完整性是可以接受的。此外，可以禁用校验和，特别是底层系统本身就支持校验和，这种情况使用RawLocalFileSystem替代LocalFileSystem。要想在一个应用中实现**全局校验和验证，**需要将fs.file.impl属性设置为org.apache.hadoop.fs.RawLocalFileSystem进而实现对文件URI的重新映射。还有一个可选方案可以直接新建一个RawLocalFileSystem实例。如果想针对一些读操作禁用校验和，这个方案非常有用。示例如下：

```java
Configuration conf = ...
FileSystem fs = new RawLocalFileSystem();
fs.initialize(null, conf);
```

### 5.1.3 ChecksumFileSystem

LocalFileSystem通过ChecksumFileSystem来完成自己的任务，向其他文件系统(无校验和系统)加入校验和非常简单，因为ChecksumFileSystem类继承自FileSystem类。一般用法如下：

```java
FileSystem rawFs = ......
FileSystem checksummedFs = new ChecksumFileSystem(rawFs);
```

底层文件系统称为“源(raw)”文件系统，可以使用ChecksumFileSystem实例的getRawFileSystem( )方法获取。ChecksumFileSystem类还有其他一些与校验和有关的有用方法，比如getChecksumFile()可以获得任意一个文件的校验和文件路径。

如果ChecksumFileSystem类在读取文件时检测到错误，会调用自身reportChecksumFailure( )方法，默认实现方法为空。但LocalFileSystem类会将这个出错的文件及其校验和移到同一存储设备上一个名为*bad_files*的边际文件夹(side directory)中。管理员应该定期检查这些坏文件并采取相应的行动。

## 5.2 压缩

文件压缩有两大好处：

- 减少存储文件所需要的磁盘空间。
- 加速数据在网络和磁盘上的传输速度。

有很多种不同的压缩格式、工具和算法，它们各有千秋。表5-1列出了与Hadoop结合使用的常见压缩方法。

​															**表5-1. 压缩格式总结**

| 压缩格式 | 工具  | 算法    | 后缀名   | 是否可切分 |
| -------- | ----- | ------- | -------- | ---------- |
| DEFLATE  | 无    | DEALATE | .deflate | 否         |
| gzip     | gzip  | DEFLATE | .gz      | 否         |
| bzip2    | bzip2 | bzip2   | .bz2     | 是         |
| LZO      | lzop  | LZO     | .lzo     | 否         |
| LZ4      | 无    | LZ4     | .lz4     | 否         |
| Snappy   | 无    | Snappy  | .snappy  | 否         |

**注意：**DEFLATE是一个标准压缩算法，该算法的标准实现是zlib。没有可用于生成DEFLATE文件的常用命令行工具。因为通常都用gzip格式。注意，gzip文件格式只是在DEFLATE格式上增加了文件头和一个文件尾。.*deflate*文件扩展名是Hadoop约定的。

所有压缩算法都需要权衡空间/时间：压缩和解压缩速度更快，其代价通常是只能节省少量的空间。表5-1列出的所有压缩工具都提供9个不同的选项来控制压缩时必须考虑的权衡：选项-1为优化压缩速度，-9为优化压缩空间。例如，下述命令通过最快的压缩方法创建一个名为*file.gz*的压缩文件：

```sh
gzip -1 file
```

不同的压缩工具有不同的特性：

- gzip是一个通用的压缩工具，在空间/时间性能的权衡中，居于其他两个压缩方法之间。
- bzip2的压缩能力强于gzip，但压缩速度更慢一点。尽管bzip2的解压速度比压缩速度快，但仍比其他压缩格式要慢一些。
- LZO、LZ4和Snappy均优化压缩速度，其速度比gzip快一个数量级，但压缩效率稍逊一筹。Snappy和LZ4的解压缩速度比LZO高出很多。

表5-1中的“是否可切分”列表示对应的压缩算法是否支持切分(splitable)，即是否可以搜索数据流的任意位置并进一步往下读取数据。可切分压缩格式尤其适合MapReduce，更多讨论，可以参见5.2.2节。

### 5.2.1 codec

codec是压缩-解压缩算法的一种实现。在Hadoop中，一个对CompressionCodec接口的实现代表一个codec。例如，**GzipCodec**包装了gzip的压缩和解压缩算法。表5-2列举了Hadoop实现的codec。

​													**表5-2. Hadoop的压缩codec**

| 压缩格式 | HadoopCompressionCodec                     |
| -------- | ------------------------------------------ |
| DEFLATE  | org.apache.hadoop.io.compress.DefaultCodec |
| gzip     | org.apache.hadoop.io.compress.GzipCodec    |
| bzip2    | org.apache.hadoop.io.compress.Bzip2Codec   |
| LZO      | org.apache.hadoop.io.compress.LzopCodec    |
| LZ4      | org.apache.hadoop.io.compress.LZ4Codec     |
| Snappy   | org.apache.hadoop.io.compress.SnappyCodec  |

**注意**：LZO代码库拥有GPL许可，因而可能没有包含在Apache的发行版本中，因此，Hadoop的codec需要单独从Google(http://code.google.com/p/hadoop-gpl-compression)或GitHub(http://github.com/kevinweil/hadoop-lzo)下载。LzopCodec与lzop工具兼容，LzopCodec基本上是LZO格式的但包含额外的文件头，也有针对纯LZO格式的LzoCodec，并使用*.lzo_deflate*作为文件扩展名(类似于DEFLATE，是gzip格式但不包含文件头)。

**1. 通过CompressionCodec对数据流进行压缩和解压缩**

CompressionCodec包含两个函数：

- createOutptStream(OutputStream out)：对写入输入的数据流压缩，在底层的数据流中对需要以压缩格式写入在此之前尚未压缩压缩的数据新建一个CompressionOutputStream对象。
- createInputStream(InputStream in)：对输入数据流中读取的数据进行解压缩时，调用该方法获取CompressionInputStream对象，从底层数据流读取解压缩后的数据。

CompressionOutputStream和CompressionInputStream，类似于java.util. zip.DeflaterOutputStream和java.util.zip.DeflaterInputStream，只不过前两者能够重置其底层的压缩或解压缩方法，对于某些将部分数据流(section of data stream)压缩为单独数据块(block)的应用。

**范例5-1. 该程序压缩从标准输入读取的数据，然后将其写到标准输出**

``` java
public class StreamCompressor {
  
  public static void main(String[] args) throws Exception {
    String codecClassname = args[0];
    Class<?> codecClass = Class.forName(codecClassname);
    Configuration conf = new Configuration();
    CompressionCodec codec = (CompressionCodec);
    // 使用ReflectionUtils新建一个codec实例，并由此获得在System.out上支持压缩的一个包裹方法。
    ReflectionUtils.newInstance(codecClass, conf);
    CompressionOutputStream out = codec.createOutputStream(System.out);
    // 对IOUtils对象调用copyBytes()方法将输入数据复制到输出，输出由CompressionOutputStream对象压缩
    IOUtils.copyBytes(System.in, out, 4096, false);
    //CompressionOutputStream对象调用finish()方法，要求压缩方法完成到压缩数据流的写操作，但不关闭数据流
    out.finish();
  }
  
}
```

通过GzipCodec的StreamCompressor对象对字符串“Text”进行压缩，然后使用*gunzip*从标准输入中对它进行读取并解压缩操作：

```
hadoop StreamCompressor org.apache.hadoop.io.compress.GzipCodec \
gunzip Text
```

**2.  通过CompressionCodeFactory 推断CompressionCodec**

在读取一个压缩文件时，通常可以通过文件扩展名推断需要使用哪个codec。如果文件以*.gz*结尾，则可以用GzipCodec来读取，如此等等。前面的表5-1为每一种压缩格式列举了文件扩展名。

通过使用CompressionFactory的getCodec()方法，可以将文件扩展名映射到一个CompressionCodec的方法，该方法取文件的Path对象作为参数。

**范例5-2. 该应用根据文件扩展名选取codec解压缩文件**

```java
public class FileDecompressor {
  
  public static void main(String[] args) throws Exception {
    String uri = args[0];
    Configuration conf = new Configuration();
    FileSystem fs = FileSystem.get(URI.create(uri), conf);
    Path inputPath = new Path(uri);
    CompressionCodecFactory factory = new CompressionCodecFactory(conf);
    CompressionCodec codec = factory.getCodec(inputPath);
    if (codec == null) {
      System.err.println("No codec found for " + uri);
      System.exit(1);
    }
    // 去除文件扩展名形成输出文件名
    String outputUri =
				CompressionCodecFactory.removeSuffix(uri, codec.getDefaultExtension());
    InputStream in = null;
    OutputStream out = null;
    try {
      in = codec.createInputStream(fs.open(inputPath));
      out = fs.create(new Path(outputUri));
      IOUtils.copyBytes(in, out, conf);
    } finally {
      IOUtils.closeStream(in);
      IOUtils.closeStream(out);
    }
  }
  
}
```

按照这种方法，一个名为*file.gz*的文件可以通过调用该程序解压为名为*file*的文件：

```sh
hadoop FileDecompressor file.gz
```

CompressionCodecFactory加载表5-2除LZO之外的所有codec，同样也加载io.compression.codecs配置属性(参见表5-3)列表中的所有codec。在默认情况下，该属性列表是空的，可能只有在拥有一个希望注册的定制codec(例如外部管理的LZO codec)时才需要加以修改。

​															**表5-3. 压缩codec的属性**

| 属性名                | 属性           | 默认值 | 描述                                                  |
| --------------------- | -------------- | ------ | ----------------------------------------------------- |
| io.compression.codecs | 逗号分隔的类名 | 空     | 用于压缩/解压缩的额外自定义的CompressionCodec类的列表 |

**3. 原生类库**

为了提高性能，最好使用原生 (native)类库来实现压缩和解压缩。例如：在一个测试中，使用原生gzip类库与内置的Java实现相比可以减少约一半的解压缩时间和约10%的压缩时间。

​															**表5-4. 压缩代码库的实现**

| 压缩格式 | 是否有Java实现 | 是否有原生实现 |
| -------- | -------------- | -------------- |
| DEFLATE  | 是             | 是             |
| gzip     | 是             | 是             |
| bzip2    | 是             | 否             |
| LZO      | 否             | 是             |
| LZ4      | 否             | 是             |
| Snappy   | 否             | 是             |

可以通过Java的java.library.path属性指定原生代码库，HADOOP_HOME/ect/hadoop中的hadoop脚本可以设施该属性；也可以在代码中手动设置；

默认情况下，Hadoop会根据自身运行的平台搜索原生代码库，如果找到相应的代码库就会自动加载。这意味着，无需为了使用原生代码库而修改任何设置。但是，在某些情况下，例如调试一个压缩相关问题时，可能需要禁用原生代码库。将属性io.native.lib.available的值设置成false即可，这可确保使用内置的Java代码库(如果有的话)。

**4. CodecPool**

如果使用的是原生代码库并且需要在应用中执行大量压缩和解压缩操作，可以考虑使用CodecPool，它支持反复使用压缩和解压缩，以分摊创建这些对象的开销。

**范例5-3. 使用压缩池对读取自标准输入的数据进行压缩，然后将其写到标准输出**

```java
public class PooledStreamCompressor {
  
    public static void main(String[] args) throws Exception {
      String codecClassname = args[0]; 
      Class<?> codecClass = Class.forName(codecClassname);
	    Configuration conf = new Configuration();
	    CompressionCodec codec = (CompressionCodec)ReflectionUtils.newInstance(codecClass, conf);
      Compressor compressor = null;
      try {
        compressor = CodecPool.getCompressor(codec);
	      CompressionOutputStream out = codec.createOutputStream(System.out, compressor);
	      IOUtils.copyBytes(System.in, out, 4096, false);
	      out.finish();
    	} finally {
        // 在不同的数据流之间来回复制数据，出现异常时，则确保compressor对象返回池中
      	CodecPool.returnCompressor(compressor);
    	}
    }  
}
```

### 5.2.2 压缩和输入分片

在考虑如何压缩将由MapReduce处理数据时，压缩格式是否支持切分(splitting)十分重要。以一个存储在HDFS文件系统中且压缩前大小为1 GB的文件为例。如果HDFS的块大小设置为128 MB，那么该文件将被存储在8个块中，把这个文件作为输入数据的MapReduce作业，将创建8个输入分片，其中每个分片作为一个单独的map任务的输入被独立处理。

假设文件经过gzip压缩，且压缩后文件大小为1GB，与此前一样，HDFS将这个文件保存为8个数据块。但是，将每个数据块单独作为一个输入分片是无法实现工作的，因为无法实现从gizp压缩数据流的任意位置读取数据，所以让map任务独立于其他任务进行数据读取是行不通的。gzip格式使用DEFLATE算法来存储压缩后的数据，而DEFLATE算法将数据存储在一系列连续的压缩块中。每个块的起始位置没有任何形式的标记，以读取时无法从数据流的任意当前位置前进到下一块的起始位置读取下一个数据块，从而实现与整个数据流的同步。由于上述原因，**gzip并不支持文件切分**。

在这种情况下，MapReduce会采用正确的做法，不会尝试切分gzip压缩文件，因为知道输入是gzip压缩文件(通过文件扩展名看出)且gzip不支持切分。这是可行的，但**牺牲了数据的本地性**：一个map任务处理8个HDFS块，而其中大多数块并没有存储在执行该map任务的节点上。而且，map任务数越少，作业的粒度就较大，因而运行的时间可能会更长。bzip2文件提供不同数据块之间的同步标识(pi的48位近似值)，因而它支持切分。

**应该使用哪种压缩格式**

Hadoop应用处理的数据集非常大，因此需要借助压缩，使用哪种压缩格式与待处理的文件的大小、格式和所使用的工具相关。下面有一些建议，大致是按照效率从高到低排列的。

- 使用容器文件格式，例如顺序文件(见5.4.1节)、Avro数据文件、ORCFiles或者Parquet文件，所有这些文件格式同时支持压缩和切分，通常最好与一个快速压缩工具联合使用，如：LZO，LZ4或者Snappy。
- 使用支持切分的压缩格式，例如bzip2(尽管bzip2非常慢)，或者使用通过索引实现切分的压缩格式，例如LZO。
- 将应用中文件切分为块，并使用任意一种压缩格式为每个数据块建立压缩文件，这种情况下，需要合理选择数据块的大小，以确保压缩后数据块的大小近似于HDFS块的大小。
- 存储未压缩文件

**对于大文件来说，不要使用不支持切分和压缩的文件格式，因为会失去数据的本地特性，进而造成MapReduce应用效率低下**。

### 5.2.3 在MapReduce中使用压缩

前面讲到通过CompressionCodecFactory来推断CompressionCodec时指出，如果输入文件是压缩的，那么在根据文件扩展名推断出相应的codec后，**MapReduce会在读取文件时自动解压缩文件**。

若要压缩MapReduce作业的输出，有两种配置方案：

- 在作业配置过程中将mapreduce.output.fileoutputformat.compress属性设置为true，将map reduce.output.fileoutputformat.compress.codec属性设置为要使用的压缩codec类名。
- 在FileOutputFormat中使用更便捷的方法设置这些属性，如范例5-4所示。

**范例5-4. 对查找最高气温作业所产生输出进行压缩**

```java
public class MaxTemperatureWithCompression {
  
   public static void main(String[] args) throws IOException {
     if (args.length != 2) {
     	System.err.println("Usage: MaxTemperatureWithCompression <input path>" + "<output path>");
      System.exit(-1);
     }
     Job job = new Job();
     FileInputFormat.addInputPath(job, new Path(args[0]));
     FileOutputFormat.addOuputPath(job, new Path(args[1]));
     job.setOutputKey(Text.class);
     job.setOutputValueClass(IntWritable.class);
     FileOutputFormat.setCompressOutput(job, true);
     FileOutputFormat.setOutputCompressorClass(job, GzipCodec.class);
     job.setMapperClass(MaxTemperatureMapper.class);
     job.setCombinerClass(MaxTemperatureReducer.class);
     job.setReducerClass(MaxTemperatureReducer.class);
     System.exit(job.waitForCompletion(true) ? 0 : 1);
   }
  
}
```

按照如下指令对压缩后的输入运行程序：

```
 hadoop MaxTemperatureWithCompression input/ncdc/sample.txt.gz output
```

如果为输出生成顺序文件(sequence file)，可以设置mapreduce.out put.fileoutputformat.compress.type属性来控制限制使用压缩格式。默认值是RECORD，即针对每条记录进行压缩。如果将其改为BLOCK，将针对一组记录进行压缩，这是推荐的压缩策略，因为它的压缩效率更高(参见5.4.1节)。SequenceFileOutputFormat类另外还有一个静态方法putCompressionType()，可以用来便捷地设置该属性。

表5-5归纳概述了用于设置MpaReduce作业输出的压缩格式的配置属性。如果MapReduce驱动使用Tool接口(参见6.2.2节)，则可以通过命令行将这些属性传递给程序，这比通过程序代码来修改压缩属性更加简便。

​														**表5-5. MapReduce的压缩属性**

| 属性名                                           | 类型    | 默认值                                      | 描述                                                  |
| ------------------------------------------------ | ------- | ------------------------------------------- | ----------------------------------------------------- |
| mapreduce.output.fileoutputformat.compress       | boolean | false                                       | 是否压缩输出                                          |
| mapreduce.output.fileoutputformat.compress.codec | 类名称  | org.apache.hadoop.io. compress.DefaultCodec | map输出所用的压缩codec                                |
| mapreduce.output.fileoutputformat.compress.type  | String  | RECORD                                      | 顺序文件输出可以使用的压缩类型：NONE、RECORD或者BLOCK |

**对map任务输出进行压缩**

尽管mapreduce应用读/写的未经压缩的数据，但如果对map阶段的中间结果进行压缩，可以获得不少好处。由于map任务的输出需要写到磁盘文件并通过网络传输到redcuer节点，所以通过使用LZO、LZ4或者Snappy这样的快速压缩方式，可以获得性能的提升，因为需要传输的数据减少了，启用map任务输出压缩和设置压缩格式的配置属性如表5-6所示。

​													**表5-6. map任务输出的压缩属性**

| 属性名                              | 类型    | 默认值                                      | 描述                      |
| ----------------------------------- | ------- | ------------------------------------------- | ------------------------- |
| mapreduce.map.output.compress       | boolean | false                                       | 是否对map任务输出进行压缩 |
| mapreduce.map.output.compress.codec | class   | org.apache.hadoop.io. compress.DefaultCodec | map输出所用的压缩codec    |

在作业中启用map任务输出gzip压缩格式的代码：

``` java
Configuration conf = new Configuration();
conf.setBoolean(Job.MAP_OUTPUT_COMPRESS, true);
conf.setClass(Job.MAP_OUTPUT_COMPRESS_CODEC, GzipCodec.class,CompressionCodec.class);
Job job = new Job(conf);
```

