# 附录C 准备NCDC气象数据

首先说明如何处理原始的气象文件。原始数据实际上是一组胫骨bzip2压缩的tar文件，每个年份的数据单独放在一个文件中。部分文件列举如下：

```
1901.tar.bz2  
1902.tar.bz2  
1903.tar.bz2  
...  
2000.tar.bz2 
```

各个tar文件包含一个gzip压缩文件，描述某一年度所有气象站的天气记录。(事实上，由于在存档中的各个文件已经预先压缩过，因此再利用bzip2对存档压缩就稍显多余了)。示例如下：

```sh
tar jxf 1901.tar.bz2  
ls -l 1901 | head  
011990-99999-1950.gz  
011990-99999-1950.gz  
...  
011990-99999-1950.gz 
```

由于气象站数以万计，所以整个数据集实际上是由大量小文件构成的。鉴于Hadoop对少量的大文件的处理更容易、更高效(参见7.2.1节)，所以在本例中，我们将每个年度的数据解压缩到一个文件中，并以年份命名。上述操作可由一个MapReduce程序来完成，以充分利用其并行处理能力的优势。下面具体看看这个程序。

范例c-1 利用bash脚本来处理原始的NCDC数据文件并建将其存储在HDFS中

```bash
#!/usr/bin/env bash
# NLineInputFormat gives a single line: key is offset, value is S3 URI
read offset s3file

# Retrieve file from S3 to local disk
echo "reporter:status:Retrieving $s3file" >&2
$HADOOP_HOME/bin/hadoop fs -get $s3file

# Un-bzip and un-tar the local file
target=`basename $s3file .tar.bz2`
mkdir -p $target
echo "reporter:status:Un-tarring $s3file to $target" >&2

# Un-gzip each station file and concat into one file
echo "reporter:status:Un-gzipping $target" >&2
for file in $target/*/*
do
	gunzip -c $file >> $target.all
	echo "reporter:status:Processed $file" >&2
Done

# Put gzipped version into HDFS
echo "reporter:status:Gzipping $target and putting in HDFS" >&2
gzip -c $target.all | $HADOOP_HOME /bin/hadoop fs -put - gz/$target.gz
```

输入是一个小的文本文件(ncdc_files.txt)，列出了所有待处理文件(这些文件放在S3文件系统中，因此能够以Hadoop所认可的S3 URI的方式被引用)。示例如下：

```
s3n://hadoopbook/ncdc/raw/isd-1901.tar.bz2  
s3n://hadoopbook/ncdc/raw/isd-1902.tar.bz2  
...  
s3n://hadoopbook/ncdc/raw/isd-2000.tar.bz2 
```

通过将输入格式指定为NLineInputFormat，每个mapper接受一行输入(包含必须处理的文件)。处理过程在脚本中解释，但简单说来，它会解压缩bzip2文件，然后将该年份所有文件整合为一个文件。最后，该文件以gzip进行压缩并复制至HDFS之中。注意，使用指令hadoop fs –put - 能够从标准输入中，获得数据。

状态消息输出到“标准错误”(以reporter:status为前缀)，可以解释为MapReduce状态更新。这告诉Hadoop该脚本正在运行，并未挂起。

运行Streaming作业的脚本如下：

```sh
% hadoop jar $HADOOP_INSTALL/contrib/streaming/hadoop-*-streaming.jar \  
  -D mapred.reduce.tasks=0 \  
  -D mapred.map.tasks.speculative.execution=false \  
  -D mapred.task.timeout=12000000 \  
  -input ncdc_files.txt \  
  -inputformat org.apache.hadoop.mapred.lib.NLineInputFormat \  
  -output output \  
  -mapper load_ncdc_map.sh \  
  -file load_ncdc_map.sh 
```

这是一个“只有map”的作业，因为reduce任务数为0。以上脚本还关闭了推测执行(speculative execution)，因此重复的任务不会写相同的文件(6.5.3节所讨论的方法也是可行的)。任务超时参数被设置为一个比较大的值，使得Hadoop不会杀掉那些运行时间较长的任务，例如，在解档文件或将文件复制到HDFS时，或者当进展状态未被报告时。
