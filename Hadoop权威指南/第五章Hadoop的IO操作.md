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

