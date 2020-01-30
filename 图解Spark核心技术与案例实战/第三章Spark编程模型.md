Spark 编程模型

## 3.3 编程接口

Spark中提供了通用接口来抽象每个RDD，这些接口包括：

- 分区信息：数据集的最小分片

- 依赖关系：指向父RDD

- 函数：基于父RDD的计算方法

- 划分策略和数据位置的元数据

	​                                 						**表3-1 RDD编程接口**

| 操作                    | 含义                                    |
| :---------------------- | :-------------------------------------- |
| Partitions()            | 返回分区对象列表                        |
| PreferredLocations(p)   | 根据数据的本地特性，列出分片p的首选位置 |
| Dependencies()          | 返回依赖列表                            |
| Iterator(p.parentIters) | 给定p的父分片的迭代器，计算分片p的元素  |
| Partitioner()           | 返回说明RDD是否Hash或范围分片的元数据   |

 ### 3.3.1 RDD分区(Partitions)

RDD划分为多个分区分布在集群结点中，分区的多少涉及对这个RDD进行并行计算的粒度。分区是一个逻辑概念，变换前后的新旧分区可能在物理上是同一块内存或硬件存储空间，这种优化防止函数式不变形导致的内存需求无限增长。默认程序分配到的CPU核数；如果从HDFS文件创建，默认文件的数据块数；亦可自定义数量。

### 3.3.2 RDD首选位置

在Spark形成任务有向无环图 DAG时，会尽可能将计算分配到靠近数据的位置，减少网络IO交互。当RDD产生时候存在首选位置，如HDFSRDD分区首选位置是HDFS块所在节点；当RDD分区被缓存，则计算应发送至缓存分区所在节点；回溯RDD“血统”找到具有首选位置属性的父RDD，决定子RDD的分区位置。

### 3.3.3 RDD的依赖关系

在RDD中将依赖分为两种类型：窄依赖于宽依赖。

<img src="https://www.2cto.com/uploadfile/Collfiles/20180210/20180210103654165.png" style="zoom:150%;" />

窄依赖指每个父RDD分区都至多被一个子RDD分区使用，宽依赖指多个子RDD分区依赖一个父RDD分区。例如Map操作是一个窄依赖。

这两种依赖的区别从两个方面来说比较有用：**第一**，窄依赖允许在单个集群节点上流水线式的执行，这个节点可以计算所有父级分区，例如可以逐个元素的执行map和filter操作。相反宽依赖需要所有的父RDD分区数据可用，并且数据通过MapReduce的shuffle完成。**第二**，在窄依赖中，节点失败后的恢复更加高效，因为只有丢失的父类分区需要重新计算，并且这些的父级分区可以并行的在不同节点上重新计算；于此相反，宽依赖的继承关系中，单个失败节点可能导致一个RDD的所有先祖RDD中的一些分区丢失，导致计算的重新执行（宽依赖的子RDD分区依赖多个父RDD分区，在节点A计算子RDD的某个分区数据失败，可能会导致父RDD的多个分区数据重新计算，即导致整个计算任务重新进行，而窄依赖是一对一的关系，只需要重新计算单个父RDD分区即可，不会影响其他分区计算）。

### 3.3.4 RDD分区计算（Iterator）

Spark中的RDD计算式已分区为单位的，而且计算函数都是在对迭代器复合，不需要保存每次计算结果，分区计算一般使用mapPartitions等操作进行，应用于每个分区，即把分区当作一个整体进行处理。

```scala
def mapPartitions[U : ClassTag](f : Iterator[T] => Iterator[U], perservesPartitioning : Boolean = false):RDD[U]
```

f即为输入函数，处理每个分区的数据，每个分区的数据以Iterator[T]传递给函数f，f的输出结果为Iterator[U]，最终RDD由所有分区经过输入函数f处理后的结果合并起来。

### 3.3.5 RDD分区函数（Partitioner）

分区划分对Shuffle类操作很关键，决定了父RDD与子RDD之间的依赖类型（窄依赖or宽依赖）。如果协同划分的话，父子RDD之间形成**一致性分区**安排，保证同一个Key被分配到同一个分区，即窄依赖。反之，则形成宽依赖。一致性分区指的是分区划分器产生分区计算前和后一致的分区安排。

Spark默认提供两种分区划分器：哈希分区划分器（HashPartitioner）和范围划分器（RangePartitioner），且Partitioner值存在于$<K,V>$类型的RDD中！！

## 3.4 创建操作

目前有两种类型的基础RDD：一种是并行集合（Paralleized Collections），接受一个已经存在的Scala集合，然后进行并行计算；另外一种是从外部存储创建RDD，外部存储可以是从文本文件或者HDFS中读取。这两种类型的RDD获取子RDD等一系列拓展，形成**“血统”**关系图。

### 3.4.1 并行化集合创建操作

并行化集合是通过SparkContext的paralleize方法，在一个已经存在的集合上创建的，集合的对象都会被复制，创建出一个可并行化操作的分布式数据集RDD。

- Java语言

	``` java
	public <T> JavaRDD<T> parallelize(List<T> list, int numSlices)；//指定分区数量
	
	public <T> JavaRDD<T> parallelize(List<T> list) {
	        return this.parallelize(list, this.sc().defaultParallelism()); //使用默认分区数量，根据本机Spark分配到的CPU核数决定
	}
	```

- Python语言

	```python
	    def parallelize(self, c, numSlices=None):
	        """
	        Distribute a local Python collection to form an RDD. Using xrange
	        is recommended if the input represents a range for performance.
	
	        >>> sc.parallelize([0, 2, 3, 4, 6], 5).glom().collect()
	        [[0], [2], [3], [4], [6]]
	        >>> sc.parallelize(xrange(0, 6, 2), 5).glom().collect()
	        [[], [0], [], [2], [4]]
	        """
	```

- Scala

  ```
  
  ```

  ### 3.4.2 外部存储创建操作

  ​		Spark可以将任何Hadoop所支持的存储资源转换为RDD，例本地文件（需要网络文件系统，所有节点都必须能访问到）、HDFS、HBase、Amazon S3等。Spark支持文本文件，SequenceFiles和任何Hadoop InputFormat格式。

  #### 3.4.2.1 Java API

  ```java
    /**
     * Read a text file from HDFS, a local file system (available on all nodes), or any
     * Hadoop-supported file system URI, and return it as an RDD of Strings.
     */
    def textFile(path: String): JavaRDD[String] = sc.textFile(path)
  ```

  ```java
   /**
     * Read a directory of text files from HDFS, a local file system (available on all nodes), or any
     * Hadoop-supported file system URI. Each file is read as a single record and returned in a
     * key-value pair, where the key is the path of each file, the value is the content of each file.
     *
     * <p> For example, if you have the following files:
     * {{{
     *   hdfs://a-hdfs-path/part-00000
     *   hdfs://a-hdfs-path/part-00001
     *   ...
     *   hdfs://a-hdfs-path/part-nnnnn
     * }}}
     *
     * Do `val rdd = sparkContext.wholeTextFile("hdfs://a-hdfs-path")`,
     *
     * <p> then `rdd` contains
     * {{{
     *   (a-hdfs-path/part-00000, its content)
     *   (a-hdfs-path/part-00001, its content)
     *   ...
     *   (a-hdfs-path/part-nnnnn, its content)
     * }}}
     *
     * @note Small files are preferred, large file is also allowable, but may cause bad performance.
     * @note On some filesystems, `.../path/&#42;` can be a more efficient way to read all files
     *       in a directory rather than `.../path/` or `.../path`
     * @note Partitioning is determined by data locality. This may result in too few partitions
     *       by default.
     *
     * @param path Directory to the input data files, the path can be comma separated paths as the
     *             list of inputs.
     * @param minPartitions A suggestion value of the minimal splitting number for input data.
     * @return RDD representing tuples of file path and the corresponding file content
     */
    def wholeTextFiles(
        path: String,
        minPartitions: Int = defaultMinPartitions): RDD[(String, String)] = withScope {
      assertNotStopped()
      val job = NewHadoopJob.getInstance(hadoopConfiguration)
      // Use setInputPaths so that wholeTextFiles aligns with hadoopFile/textFile in taking
      // comma separated files as input. (see SPARK-7155)
      NewFileInputFormat.setInputPaths(job, path)
      val updateConf = job.getConfiguration
      new WholeTextFileRDD(
        this,
        classOf[WholeTextFileInputFormat],
        classOf[Text],
        classOf[Text],
        updateConf,
        minPartitions).map(record => (record._1.toString, record._2.toString)).setName(path)
    }
  ```

  

  #### 3.4.2.2 Python API

- textFile：使用textFile可以将本地文件或HDFS文件转换为RDD，该操作支持整个文件目录的读取，文件可以是文本或压缩文件（自动执行解压缩，并加载数据）。

	```python
	def textFile(self, name, minPartitions=None, use_unicode=True):
	        """
	        Read a text file from HDFS, a local file system (available on all
	        nodes), or any Hadoop-supported file system URI, and return it as an
	        RDD of Strings.
	
	        If use_unicode is False, the strings will be kept as `str` (encoding
	        as `utf-8`), which is faster and smaller than unicode. (Added in
	        Spark 1.2)
	
	        >>> path = os.path.join(tempdir, "sample-text.txt")
	        >>> with open(path, "w") as testFile:
	        ...    _ = testFile.write("Hello world!")
	        >>> textFile = sc.textFile(path)
	        >>> textFile.collect()
	        [u'Hello world!']
	        """
	```

- wholeTextFiles：读取文件目录中的小文件，返回（文件名，文件内容）键值对。

	```python
	    def wholeTextFiles(self, path, minPartitions=None, use_unicode=True):
	        """
	        Read a directory of text files from HDFS, a local file system
	        (available on all nodes), or any  Hadoop-supported file system
	        URI. Each file is read as a single record and returned in a
	        key-value pair, where the key is the path of each file, the
	        value is the content of each file.
	
	        If use_unicode is False, the strings will be kept as `str` (encoding
	        as `utf-8`), which is faster and smaller than unicode. (Added in
	        Spark 1.2)
	
	        For example, if you have the following files::
	
	          hdfs://a-hdfs-path/part-00000
	          hdfs://a-hdfs-path/part-00001
	          ...
	          hdfs://a-hdfs-path/part-nnnnn
	
	        Do C{rdd = sparkContext.wholeTextFiles("hdfs://a-hdfs-path")},
	        then C{rdd} contains::
	
	          (a-hdfs-path/part-00000, its content)
	          (a-hdfs-path/part-00001, its content)
	          ...
	          (a-hdfs-path/part-nnnnn, its content)
	
	        .. note:: Small files are preferred, as each file will be loaded
	            fully in memory.
	
	        >>> dirPath = os.path.join(tempdir, "files")
	        >>> os.mkdir(dirPath)
	        >>> with open(os.path.join(dirPath, "1.txt"), "w") as file1:
	        ...    _ = file1.write("1")
	        >>> with open(os.path.join(dirPath, "2.txt"), "w") as file2:
	        ...    _ = file2.write("2")
	        >>> textFiles = sc.wholeTextFiles(dirPath)
	        >>> sorted(textFiles.collect())
	        [(u'.../1.txt', u'1'), (u'.../2.txt', u'2')]
	        """
	```

- SequenceFile：SequenceFile[K,V]可以将SequenceFile转换为RDD，SequenceFile文件是Hadoop用来存储二进制形式的key-value对而设计的一种文本文件。

	```python
	    def sequenceFile(self, path, keyClass=None, valueClass=None, keyConverter=None,
	                     valueConverter=None, minSplits=None, batchSize=0):
	        """
	        Read a Hadoop SequenceFile with arbitrary key and value Writable class from HDFS,
	        a local file system (available on all nodes), or any Hadoop-supported file system URI.
	        The mechanism is as follows:
	
	            1. A Java RDD is created from the SequenceFile or other InputFormat, and the key
	               and value Writable classes
	            2. Serialization is attempted via Pyrolite pickling
	            3. If this fails, the fallback is to call 'toString' on each key and value
	            4. C{PickleSerializer} is used to deserialize pickled objects on the Python side
	
	        :param path: path to sequncefile
	        :param keyClass: fully qualified classname of key Writable class
	               (e.g. "org.apache.hadoop.io.Text")
	        :param valueClass: fully qualified classname of value Writable class
	               (e.g. "org.apache.hadoop.io.LongWritable")
	        :param keyConverter:
	        :param valueConverter:
	        :param minSplits: minimum splits in dataset
	               (default min(2, sc.defaultParallelism))
	        :param batchSize: The number of Python objects represented as a single
	               Java object. (default 0, choose batchSize automatically)
	        """
	```

- HadoopFile：由于Hadoop的接口有新旧两个版本，所以Spark为了兼容Hadoop的版本，也提供了两套接口：newAPIHadoopFile和hadoopFile。

	```scala
	def newAPIHadoopFile[K, V, F <: NewInputFormat[K, V]](
	      path: String,
	      fClass: Class[F],
	      kClass: Class[K],
	      vClass: Class[V],
	      conf: Configuration = hadoopConfiguration): RDD[(K, V)] = withScope {
	    assertNotStopped()
	```

- HadoopRDD与newAPIHadoopRDD：针对新旧两个版本Hadoop，将Hadoop输入类型转换为RDD，每个HDFS数据块成为一个RDD分区。

	```scala
	def newAPIHadoopRDD[K, V, F <: NewInputFormat[K, V]](
	      conf: Configuration = hadoopConfiguration,
	      fClass: Class[F],
	      kClass: Class[K],
	      vClass: Class[V]): RDD[(K, V)] = withScope {
	    assertNotStopped()
	
	    // This is a hack to enforce loading hdfs-site.xml.
	    // See SPARK-11227 for details.
	    FileSystem.getLocal(conf)
	
	    // Add necessary security credentials to the JobConf. Required to access secure HDFS.
	    val jconf = new JobConf(conf)
	    SparkHadoopUtil.get.addCredentials(jconf)
	    new NewHadoopRDD(this, fClass, kClass, vClass, jconf)
	  }
	```

## 3.5 转换操作

### 3.5.1 基础转换操作

- map[U] (f : (T) => U) : RDD[U] —— 对RDD中的每个元素都执行一次函数f，原RDD中的元素在新RDD中有且只有一个。

- distinct( ): RDD[ (T) ]——去除RDD中的重复元素，返回所有元素不重复的RDD。

- distinct(numPartitions : Int) : RDD[T] ——去除RDD中的重复元素，返回指定分区数的RDD。

- flatMap[U] (f : (T) => Traversable[U] ): RDD[U]——与map操作类似，区别是flatMap操作中的原RDD每个元素可以返回多个元素构成新RDD。

- coalesce(numPartitions : Int, shuffle : Boolean = false) : RDD[T]

- repartition(numPartitions : Int) : RDD[T]——coalesce和repartition都是对RDD重新分区，coalesce操作默认使用HashPartitions进行重分区，第一个参数是分区数目，第二个参数是为是否洗牌（shuffle）默认false。repartition操作是coalesce函数第二个参数为true的实现。

- randomSplit(weights : Array[Double], seed : utils.random.nextLong) : Array[RDD[T])——根据权重weights将一个RDD分割为多个RDD，注意权重总和为1，会按照权重比例划分，权重越高被划分的几率越高。

- glom( ) : RDD[Array[T]] ——glom操作是RDD中每个分区所有类型为T的数组转变成元素类型为T的数组。

	```shell
	// 定义3个分区的RDD，使用glom操作将每个分区中的元素放到一个数组中
	scala> var rdd = sc.makeRDD(1 to 10, 3)
	scala> rdd.glom().collect
	res0: Array[Array[Int]] = Array(Array(1, 2, 3), Array(4, 5, 6), Array(7, 8, 9, 10))
	```

- Union(other: RDD[T]): RDD[T]——将两个RDD合并，返回两个RDD的并集，返回元素不去重。

- intersection(other: RDD[T]): RDD[T]

- intersection(other: RDD[T], numPartitions: Int): RDD[T]

- intersection(other: RDD[T], partitioner: Partitioner): RDD[T]

	intersection操作类似于SQL中的innerJoin操作，返回两个元素交集，返回结果去重；numPartitions指定分区数，Partitoner指定分区方式。

- subtract(other: RDD[T]): RDD[T]

- subtract(other: RDD[T], numPartitions: Int): RDD[T]

- subtract(other: RDD[T], partitioner: Partitioner): RDD[T]

	subtract操作返回在RDD中出现，而otherRDD中未出现的元素，返回结果不去重。

- mapPartitons[U] (f : (Iteratot[T] => Iterator[U], preservesPartitioning: Boolean = false)): RDD[U]

- mapPartitionsWitIndex[U] (f :(index : Int, Iteratot[T] => Iterator[U], preservesPartitioning: Boolean = false)): RDD[U]

	mapPartitons操作与map操作类似，不同的是映射参数由RDD中的每个元素编程了RDD中每个分区的迭代器。参数preservesPartitioning表示是否保留父RDD的partitioner分区信息。如果在mapper过程中需要频繁创建额外对象，推荐使用mapPartitons操作，例如：对RDD分区使用数据库链接，Json解析器重用(Spark快速大数据分析)。mapPartitionsWitIndex输入参数多了RDD的各个分区索引。

- zip[U] (other: RDD[U]): RDD[(T, U)]

- zipPartitions[B, V] (rdd2: RDD[B])(f: (Iterator[T], Iteratror[B]) => Iterator[V] ): RDD[V]

- zipPartitions[B, V] (rdd2: RDD[B], preservesPartitioning)(f: (Iterator[T], Iteratror[B]) => Iterator[V] ): RDD[V]

- zipPartitions[B, ,C, V] (rdd2: RDD[B], rdd3: RDD[C], preservesPartitioning: Boolean)(f: (Iterator[T], Iteratror[B], Iteratror[C]) => Iterator[V] ): RDD[V]

- zipPartitions[B, C, D, V] (rdd2: RDD[B], rdd3: RDD[C], rdd4: RDD[D])(f: (Iterator[T], Iteratror[B], Iteratror[C], Iteratror[D], Iteratror[V]) => Iterator[V] ): RDD[V]

- zipPartitions[B, C, D, V] (rdd2: RDD[B], rdd3: RDD[C], rdd4: RDD[D], preservesPartitioning: Boolean)(f: (Iterator[T], Iteratror[B], Iteratror[C], Iteratror[D], Iteratror[V]) => Iterator[V] ): RDD[V]

	zip操作用于将两个RDD组合成Key/Value形式的RDD（PairRDD），这里默认两个RDD的分区数与分区的元素数量相同。zipPartitons操作将多个RDD按照Partition组合成为新的RDD，要求分区数量一致，分区内元素数量无要求。

- zipWithIndex( ): RDD[(T, Long)]

- zipWithUniqueld( ): RDD[(T, Long)]

	zipWithIndex操作将RDD中的元素和这个元素在RDD中的ID(索引号)组合成键值二元组对返回；zipWithUniqueld操作将RDD中的元素和一个唯一ID组合成键/值对，唯一ID生成算法如下：1. 每个分区中第一个元素的唯一ID为该分区索引号；2. 每个分区中第n个元素的唯一ID为：前一个元素的唯一ID + 该RDD的总分区数。

	zipWithIndex需要启动一个Spark作业来计算每个分区开始的索引号，而zipWithUniqueld不需要。

## 3.5.2 键值转换操作





