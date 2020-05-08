# HDFS 数据一致性

## 1. HDFS数据一致性场景

HDFS在修改(在现有文件追加)和上传文件时需要保证文件与其复本的数据一致性。本质上是分布式系统通过数据副本机制保证高可用性和分区容错性，那么就是带来如下问题：

- 当主数据发生修改或新增数据时，如何保证主数据与其复本数据一致性？
- 当并发写入时，如何保证数据一致性？

## 2. HDFS元数据的一致性

### 2.1 fsimage与edits log

客户端上传文件时，NameNode首先往edits log文件中记录元数据的操作日志。与此同时，NameNode将会在磁盘做一份持久化处理（fsimage文件）：它跟内存中的数据是对应的，如何保证和内存中的数据的一致性呢？

在edits logs满之前对内存和fsimage的数据做同步，只需要合并edits logs和fsimage上的数据即可，然后edits logs上的数据即可清除。而当edits logs满之后，文件的上传不能中断，所以将会往一个新的文件edits.new上写数据， 而老的edits logs的合并操作将由secondNameNode来完成，即所谓的checkpoint操作。

### 2.2 checkPoint

会下以下两种情况下执行checkPoint：

- edits.log文件大小限制，由`fs.checkpoint.size`配置，当edits.log文件达到上限，则会触发checkpoint操作
- 指定时间内进行checkpoint操作，由`fs.checkpoint.period`配置

无论是文件大小还是checkppoint时间窗，只要满足触发机制，都会强制执行checkpoint操作。为了不中断HDFS集群对外提供服务，checkpoint操作会在SecondarNameNode上进行。

### 2.3 SecondaryNameNode

从NameNode上下载元数据信息（fsimage、edits），然后把二者合并，生成新的fsimage，在本地保存，并将其推送到NameNode，替换旧的fsimage。

**注意：**SecondaryNameNode 只存在于Hadoop1.0中，Hadoop2.0以上版本中没有，Hadoop 2.0通过NFS与QJM在activeNameNode与standbyNode之间同步元数据，实现master节点的HA，但在伪分布模式中是有SecondaryNameNode的，在集群模式中是没有SecondaryNameNode的

checkpoint在SecondaryNameNode上执行过程如下：

- secondary通知namenode切换edits文件；
- secondary从namenode获得fsimage和edits(通过http)；
- secondary将fsimage载入内存，然后开始合并edits
- secondary将新的fsimage发回给namenode
- namenode用新的fsimage替换旧的fsimage；

## 3. HDFS文件数据一致性

HDFS通过租约机制、checksum、一致性模型来保证文件数据的一致性。

### 3.1 CheckSum

HDFS会对写入的所有数据计算校验和，并在读取数据时验证检验和，针对每个由`dfs.bytes-per-checksum`指定字节的数据计算校验和，默认为512个字节。

datanode负责在收到数据后存储该数据及其校验和之前对数据进行验证。它在收到客户端的数据或复制其他datanode的数据时执行这个操作。正在写数据的客户端将数据及其校验和发送到由一些列datanode组成的管线(pipeline)，管线中的最后一个datanode负责验证校验和。

客户端从datanode读取数据时，也会验证校验和，将他们与datanode中存储的校验和比较，每个datanode均持久保存有一个用于验证的校验和日志，所以知道每个数据块的 最后一次验证时间。客户端成功验证一个数据块后，会告诉这个datanode，datanode更新日志，保存这些统计信息对检测损坏磁盘很有价值。

由于HDFS存储着每个数据块的复本，因此可以通过数据复本来修复损坏的数据块，进而得到一个新的、完好无损的复本。基本思路是：客户端在**读取数据时**，如果检测到错误，首先向namenode报告损坏的数据块以及正在尝试读取操作的这个datanode，再抛出ChecksumException异常，namenode将这个数据复本块标记为已损坏，不再将客户端处理请求直接发送到这个节点，或尝试将这个复本复制到另一个datanode。

### 3.2 租约机制

在linux中，为了防止多个进程向同一个文件写数据的情况，采用了文件加锁的机制。而在HDFS中，同样需要一个机制来防止同一个文件被多个人写入数据。这种机制就是租约(Lease)，每当写入数据之前，一个客户端必须获得namenode发放的一个租约。Namenode保证同一个文件只发放一个允许写的租约。那么就可以有效防止多人写入的情况。

HDFS还使用租约机制保证多个复本变更顺序的一致性，master节点为数据块的一个复本建立一个租约，把该复本称为主复本，主复本对复本的所有更改操作进行序列化，所有的复本都遵循这个顺序进行修改操作，因此，修改操作全局顺序由master节点选择的租约顺序决定。

### 3.3 一致性模型

文件系统的一致性模型(coherency model)描述了文件读/写的数据可见性。HDFS牺牲了一些POSIX要求，因此一些操作与你期望的可能不同。

