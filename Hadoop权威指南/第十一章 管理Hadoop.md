# 第十一章 管理Hadoop

第10章介绍如何搭建Hadoop集群，本章将关注如何保障系统平稳运行。

## 11.1 HDFS

### 11.1.1 永久性数据结构

作为管理员，深入了解namenode、辅助namenode和datanode等HDFS组件如何在磁盘上组织永久性数据结构非常重要。洞悉各文件的用法有助于进行故障诊断和故障检出。

#### 1. namenode的目录结构

运行中的namenode有如下所示目录：

```
 ${dfs.namenode.name.dir}
 |——————current
 |  |——————VERSION
 |  |——————edits_0000000000000000001-0000000000000000019
 |  |——————edits_inprogress_0000000000000000020
 |  |——————fsimage_0000000000000000000
 |  |——————fsimage_000000000000000000.md5
 |  |——————fsimage_0000000000e00000019
 |  |——————fsimage_0000000000e00000019md5 
 |  |______seen_txid
 |____in_use.lock
```

如第十章所述，dfs.namenode.name.dir属性描述了一组目录，各个目录存储着镜像内容，该机制使系统具备了一定的复原能力，特别是当其中一个目录是NFS的一个挂载时(推荐配置)。

VERSION文件是一个Java属性文件，其中包含正在运行的HDFS的版本信息。该文件一般包含以下内容：

```
#Mon Sep 29 09:54:36 BST 2014
namespaceID=1342387246
clusterID=CID-01b5c398-959c-4ea8-aae6-le0d9bd8bl42
cTime=0
storageType=NAME_NODE
blockpoolID=BP-526805057-127.0.0.1-1411980876842
layOutVersion=-57
```

属性layOutVersion是一个负整数，描述HDFS持久性数据结构(也成布局)的版本，但是该版本号与Hadoop发布包的版本号无关。只要布局变更，版本号就会递减(例如，版本号-57之后是-58)，此时，HDFS也需要升级，否则，磁盘仍使用的是旧版本的布局，新版本的namenode(或datanode)就无法正常工作。要想知道如何升级HDFS，请参见11.3.3节。

属性namespaceID是文件系统命名空间的唯一标识符，是在namenode首次格式化是创建的。clusterID是将HDFS集群作为一个整体赋予的唯一标识符，对于联邦HDFS(见3.2.4节)非常重要，这里一个集群有多个命名空间组成，每个命名空间有一个namenode管理。blockpoolID是数据块池的唯一标识符，数据块池包含了一个namenode管理的命名空间中的所有文件。

cTime属性标记了namenode存储系统的创建时间。对于刚刚格式化的存储系统，这个属性值为0；但是在文件系统升级之后，该值会更新到新的时间戳。

storageType属性说明该存储目录包含的是namenode的数据结构。

in_use.lock文件是一个锁文件，namenode使用该文件为存储目录加锁。可以避免其他namenode实例同时使用（可能会破坏）同一个存储目录的情况。

namenode的存储目录中还包含edits、fsimage和seen_txid等二进制文件。只有深人学习的工作机理，才能够理解这些文件的用途。

#### 2. 文件系统镜像和编辑日志

文件系统客户端执行写操作时(例如：创建或移动文件)，这些事务首先被记录到编辑日志中。namenode在内存中维护文件的元数据，当编辑日志被修改时，相关元数据信息也同步更新，内存中的元数据可以支持客户端的读请求。

编辑日志在概念上是单个实体，但是它体现在磁盘是的多个文件，每个文件称为一个 **段(segment)**，名称由前缀edits及后缀组成，后缀指示出该文件所包含的事务ID。任一时刻只有一个文件处于打开可写状态(前述例子中为edits_inprogress_0000000000000000020)，在每个事务完成之后，且向客户端发送成功代码前，文件都需要更新和同步，当namenode向多个目录写数据时，只有在所有写操作更新并同步到每个分布之后方可返回成功代码，以确保任何事务都不会因为机器故障而丢失。

每个fsimage文件都是文件系统元数据的一个完整的永久性检查点(前缀表示映像文件中的最后一个事务)并非每一个写操作都会更新该文件，因为fsimage是一个大型文件（甚至可高达几个GB)，如果频繁地执行写操作，会使系统运行极为缓慢。但这个特性根本不会降低系统的恢复能力，因为如果namenode发生故障，最近的fsimage文件将被载入到内存以重构元数据的最近状态，再从相关点开始，向前执行编辑日志中记录的每个事务。事实上，namenode在启动阶段正是这样做的(参见11.1.2节对安全模式的讨论)。

------

每个fsimage文件包含文件系统中的所有目录和文件inode的序列化信息。每个inode是一个文件或目录的元数据的内部描述方式。对于文件来说，包含的信息有“复本级别”(replicationlevel)、修改时间和访问时间、访问许可、块大小、组成一个文件的块等；对于目录来说，包含的信息有修改时间、访问许可和配额元数据等信息。

数据块存储在datanode中，fsimage文件并不描述datanode。取而代之的是，namenode将这种映射关系放在内存中，当datanode加入集群时，namenode向datanode索取块列表以建立映射关系，namenode还将定期征询datanode以确保它拥有最新的块映射。

------

如前所述，编辑日志会无限增长(即使物理上它是分布在多个edis文件中)，尽管这种情况对于namenode的运行没有影响，但由于需要恢复（非常长的）编辑日志中的各项事务，namenode的重启操作会比较慢。在这段时间内，文件系统将处于离线状态，这会有违用户的期望。

解决方案是运行辅助namenode，为主namenode内存中的文件系统元数据创建检查点。创建检查点的步骤如下所示(图11-1中也概略展现了前述的编辑日志和映像文件)。

（1）辅助namenode请求主namenode停止使用正在进行中的edits文件，这样新的编辑操作记录到一个新文件中，主namenode还会更新所有存储目录中的seen_txid文件；

（2）辅助namenode从主namenode获取最近的为口ge和文件（采用HTTP GET)。

（3）辅助namenode将fsiamge文件载入内存，逐一执行edits文件中的事务，创建新的合并后的fsimage文件。

（4）辅助namenode将新的fsimage文件发送回主namenode(使用HTTP PUT)，主namenode将其保存为临时的.ckpt文件。

（5）主namenode重新命名临时的fsimage文件，便于日后使用。

最终，主namenode拥有最新的fsimage文件和一个更小的正在进行中的edits文件(文件可能非空，因为在创建检查点过程中主namenode还可能收到一些编辑请求)。当namenode处在安全模式时，管理员也可调用`hdfs dfsadmin- saveNamespace`命令来创建检查点。

这个过程清晰解释了辅助namenode和主namenode拥有相近内存需求的原因（因为辅助namenode也把fsimage文件载人内存）。因此，在大型集群中，辅助namenode需要运行在一台专用机器上。

创建检查点的触发条件受两个配置参数控制：

- 通常情况下，辅助namenode每隔一小时（由dfs.namenode.checkpoint.period属性设置，以秒为单位）创建检查点；
- 从上一个检查点开始编辑日志大小已经达到100万个事务(dfs.namenode.checkpoint.txns属性设置)时，即使不到一小时，也会创建检查点，检查频率为每分钟一次(dfs.namenode.checkpoint.check.period属性设置，以秒为单位)。

------

实际用户可以使用-checkpoint选项来启动namenode，它将运行一个检查点过程以应对主namenode，这等价于运行一个辅助namenode。

------

<img src="./img/11-1.jpg" style="zoom:67%;" />

​																	**图11-1 创建检查点过程**

#### 3. 辅助namenode的目录结构

辅助namenode的检查点目录(dfs.namenode.checkpoint.dir)的布局和主namenode的检查点目录的布局相同。这种设计方案的好处是，在主namenode发生故障时（假设没有及时备份，甚至在NFS上也没有），可以从辅助namenode恢复数据。

有两种实现方法：

- 将相关存储目录复制到新的namenode中；
- 使用`-importCheckpoint`选项启动namenode守护进程，从而将辅助namenode用作新的主namenode。借助该选项，仅当`dfs.namenode.name.dir`属性定义的目录中没有元数据时，辅助namenode会从`dfs.namenode.checkpoint.dir`属性定义的目录载人最新的检查点namenode元数据。

#### 4. datanode目录结构

和namenode不同的是，datanode的存储目录是初始阶段自动创建的，不需要额外格式化。datanode的关键文件和目录如下所示：

```
${dfs.datanode.data.dir}/
|— current
| |— BP-526805057-127.0.0.1-1411980876842
| |  |__current
| |  |— VERSION
| |  |— finalized
| |  |  |----blk_1073741825
| |  |  |----blk_1073741825_1001.meta
| |  |  |----blk_1073741826
| |  |  |----blk_1073741826_1002.meta
| |  |__ rbw
| |__VERSION
|_in_use.lock
```

HDFS数据块存储在以blk_为前缀名的文件中，文件名包含了该文件存储的块的原始字节数，每个块有一个相关联的带有后缀的元数据文件。元数据文件包括头部(含版本和类型信息)和该块各区段的一系列的校验和。

每一个块属于一个数据池，每个数据块都有自己的存储目录，目录根据数据块池ID形成和(namenode的VERSION文件中的数据块池ID相同)。

当目录中数据块的数量增加到一定规模时，datanode会创建一个子目录来存放新的数据块及其元数据信息。如果当前目录已经存储了64个(通过dfs.datanode.numblocks属性设置，默认值64)数据块时，就创建一个子目录。终极目标是设计一棵高扇出的目录树，即使文件系统中的块数量非常多，目录树的层数也不多。通过这种方式，datanode可以有效管理各个目录中的文件，避免大多数操作系统遇到的管理难题，即很多(成千上万个)文件放在同一个目录之中。

如果dfs.datanode.data.dir属性指定了不同磁盘上的多个目录，那么数据块会以**轮转(round-robin)**的方式写到各个目录中。注意，同一个datanode上的每个磁盘上的块不会重复，只有不同datanode之间的块才有可能重复。

### 11.1.2 安全模式

