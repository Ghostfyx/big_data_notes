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

namenode启动时，首先将映像文件(fsimage)载入内存，并执行编辑日志(edits)中的各项编辑操作，一旦在内存中成功建立文件系统元数据的映像，则创建一个新的fsimage文件(该操作不需要借助辅助namenode)和一个空的编辑日志。在这个过程中，namenode运行在安全模式，意味着namenode的文件系统对于客户端来说是只读的。

------

严格来说，在安全模式下，只有那些访问文件系统元数据的文件系统操作是肯定成功执行的，例如显示目录列表等。对于读文件操作来说，只有集群中当前datanode上的块可用时，才能够工作。但文件修改操作（包括写、删除或重命名）均会失败。

------

需要强调的是，系统中的数据块的位置并不是由namenode维护的，而是以块列表的形式存储在datanode中(每个datanode存储的块组成的列表)。在系统的正常操作期间，namenode会在内存中保留所有块位置的映射信息。在安全模式下，各个datanode会向namenode发送最新的块列表信息，namenode了解到足够多的块位置信息之后，即可高效运行文件系统。如果namenode认为向其发送更新信息的datanode节点过少，则它会启动块复制进程，以将数据块复制到新的datanode节点。然而，在大多数情况下上述操作都是不之要的（因为实际上namenode只需继续等待更多datanode发送更新信息即可），并浪费了集群的资源。实际上，在安全模式下namenode并不向datanode发出任何块复制或块删除的指令。

如果满足 **最小复制条件(minimal replication condition)**，namenode会在30秒钟之后就退出安全模式，所谓的最小复本条件指的是在整个文件系统中有99.9％的块满足最小复本级别（默认值是1，由dfs.namenode.replication.min属性设置，参见表11-1)。

在启动一个刚刚格式化的HDFS集群时，因为系统中还没有任何块，所以namenode不会进人安全模式。

#### 进入和离开安全模式

要想查看namenode是否处于安全模式，可以运行以下dfsadmin命令：

```sh
hdfs dfsadmin -safemode get

Safe mode is ON
```

HDFS的网页界面也能够显示namenode是否处于安全模式。

​															**表11-1．安全模式的属性**

| 属性名称                            | 类型  | 默认值 | 说明                                                         |
| ----------------------------------- | ----- | ------ | ------------------------------------------------------------ |
| dfs.namenode.replication.min        | int   | 1      | 成功执行写操作所需要创建的最少复本数目（也称为最小复本级别） |
| dfs.namenode.safemode.threshold-pct | float | 0.999  | 在namenode退出安全模式之前，系统中满足最小复本级别（由dfs.namenode.replication.min定义）的块的比例。将这项值设为0更小会令namenode无法启动安全模式，设为高于1则永远不会退出安全模式 |
| dfs.namenode.safemode.extension     | int   | 30000  | 在满足最小复本条件(由dfs.namenode.safemode.threshold-pct定义）之后，namenode还需要处于安全模式的时间似毫秒为单位)。对于小型集群（几十个节点）来说，这项值可以设为0 |

有时，用户期望在执行某条命令之前namenode先退出安全模式，特别是在脚本中，使用wait选项能够达到这个目的：

```
%hdfs dfsadmin -safemde wait
# command to read or write a file
```

管理员随时可以让namenode进入或退出安全模式，这项功能在维护和升级集群时非常关键，因为需要确保数据在指定时段内是只读的。使用以下指令进人安全模式：

```sh
hdfs dfsadmin -safemde enter
```

前面提到过，namenode在启动阶段会处于安全模式。在此期间也可使用这条命令，从而确保namenode在启动完毕之后不禽开安仝模式。另一种使namenode永远处于安全模式的方法是将属性dfs.namenode.safemode.threshold-pct的值设为大于1。

运行以下指令即可使得namenode离开安全模式：

```sh
hdfs dfsadmin -safemde leave
Safe mode is OFF
```

### 11.1.3 日志审计

HDFS日志能够记录所有文件系统的访问请求，有些组织需要这项特性来进行审计。对日志进行审计是log4j在INFO级别实现的。在默认配置下，此项特性并未启用，但是通过在文件hadoop-env.sh中增加以下这行命令，很容易启动该日志审计特性：

```
export HDFS_AUDIT_LOGGER="INFO,RFAALDIT"
```

每个HDFS事件均在审计日志(pdfs-audit.log)中生成一行日志记录，下例说明如何对“/user/tom“目录执行list status命列出指定目录下的文件/目录的状态：

```
2014-09-30 21:33:30,484 INFO FSNamesystem.audit: allowed=true ugi=tom
(auth:SIMPLE) ip=/127.0.0.1 cmd=listStatus src=/user/tom dst=null
perm=null proto=rpc
```

### 11.1.4 工具

#### 1. dfsadmin工具

dfsadmin工具用途较广，既可以查找HDFS状态信息，又可在HDFS上执行管理操作。以hdfs dfsadmin形式调用，且需要超级用户权限。

表11-2列举了部分dfsadmin命令。要想进一步了解详情，可以用-help命令。

| 命令               | 说明                                                         |
| ------------------ | ------------------------------------------------------------ |
| -help              | 显示走命令的帮助，如果耒指明命令，则显示所有命令的帮助       |
| -report            | 显示文件系统的统计信息(类似于在网页界面上显示的内容)，以及所连接的各个datanode的信息 |
| -metasave          | 将某些信息存储到Hadoop日志目录中的一个文件中，包括正在被复制或删除的块的信息以及已连接的datanode列表 |
| -safemode          | 改变或查询安全模式，参见11.1.2节对安全模式的讨论             |
| -saveNamespace     | 将内存中的文件系统映像保存为一个新的fsimage文件，重置edits文件。该操作仅在安全模式下执行 |
| -fetchlmage        | 从namenode获取最新的fsimage文件，并保存为本地文件            |
| -refreshNodes      | 更新允许连接到namenode的datanode列表，参见11.3.2节对委任和解除节点的讨论 |
| -upgradeProgress   | 获取有关HDFS升级的进度信息或强制升级。参见11.3.3对升级的讨论 |
| -finalizeUpgrade   | 移除datanode和namenode的存储目录上的旧版本数据·这个操作一般在升级完成而且集群在新版本下运行正常的情况下执行。参见11.3.3节对升级的讨论 |
| -setQuota          | 设置目录的配额，即设置以该目录为根的整个目录树最多包含多少个文件和目录。这项配置能有效阻止用户创建大量小文件，从而保护namenode的内存（文件系统中的所有文件、目录和块的各项信息均存储在内存中） |
| -clrQuota          | 清理指定目录的配额                                           |
| -setSpaceQuota     | 设置目录的空间配额，以限制存储在目录树中的所有文件的总规模。分别为各用户指定有限的存储空间很有必要 |
| -clrSpaceQuota     | 清理指定的空间配额                                           |
| -refreshServiceAcl | 刷新namenode的服务级授权策略文件                             |
| -allowSnapshot     | 允许为指定的目录创建快照                                     |
| -disallowSnapshot  | 禁止为指定的目录创建快照                                     |

#### 2. 文件系统检查fsck工具

Hadoop提供fsck工具来检查HDFS中文件的健康情况，该工具会查找哪些在所有datanode中均缺失的块记忆多多或过少复本的块。如何检查某个小型集群的整个文件系统：

```sh
hdfs fsck /

.................... Status:HEALTHY
Total size:  511799225 B 
Total dirs: 10
Total files: 22
Total blocks (validated): 22 (avg. block size 23263601 B) 
Minimally replicated blocks: 22 (100.0%)
Over-replicated blocks: 0
Under-replicated blocks: 0
Mis-replicated blocks: 0
Default replication factor: 3
B Average block replication: 3.0 
Corrupt blocks: 0
Missing replicas: 
Number of data-nodes: 4
Number of racks: 1
The filesystem under path ‘/’ is HEALTHY
```

fsck然工具从给定路径(本例是文件系统的根目录)开始循环遍历文件系统的命名空间，并检查它所找到的所有文件。对于检查过的每个文件，都会打印一个点。并检查它所找到的所有文件。对于检查过的每个文件，都会打印一个点“.”。在此过程中，该工具获取文件数据块的元信息并找出问题或检查他们是否一致。注意，工具只是从namenode获取信息，并不与任何datanode进行交互，因此并不真正获取块数据。

Fsck输出文件的大部分内容都容易理解，以下仅说明部分信息。

- **过多复制的块**  指复本数超出最小复本级别的块。严格意义上讲，这并非一个大问题，HDFS会自动删除多余复本
- **仍需复制的块**  指复本数目低于最小复本级别的块。HDFS会自动为这些块创建新的复本，直到达到最小复本级别。可以调用hdfs dfsadmin -metasave指令了解正在复制的（或等待复制的）块的信息。
- **错误复制的块**    指违反块复本放置策略的块（参见3.6.2节“复本的放置”相关内容）。例如，在最小复本级别为3的多机架集群中，如果一个块的三个复本都存储在一个机架中，则可认定该块的复本放置错误，因为一个块的复本要分散在至少两个机架中，以提高可靠性。
- **损坏的块**  指所有复本均已损坏的块。如果虽然部分复本损坏，但至少还有一个复本完好，则该块就未损坏;namenode将创建新的复本，直到达到最小复本级别。
- **缺失的复本**   指在集群中没有任何复本的块。

损坏的块和缺失的块是最需要考虑的，因为这意味着数据已经丢失了。默认情况下，然猷不会对这类块进行任何操作，但也可以让执行如下某一项操作。

- **移动**  使用-move选项将受影响的文件移到HDFS的/lost+found目录。这些受影响的文件会分裂成连续的块链表，可以帮助用户挽回损失。
- **删除**   使用-delete选项删除受影响的文件。记住，在删除之后，这些文件无法恢复。

**查找一个文件的数据块** fsck工具能够帮助用户轻松找到属于特定文件的数据块。例如：

```sh
hdfs fsck /usr/tom/part-0007 -files -blocks  -racks

/user/tom/part-00007 25582428 bytes, 1 block(s): OK
0. blk_-B724870485760122836-1035 len=25582428 repl=3 [/default-rack/10.251.43.2:50010,
/default-rack/10.251.27.178:50010, /default-rack/10.251.123.163:50010] 
```

输出内容表示文件/user/tom/part-00007包含一个块，该块的三个复本存储在不同datanode。所使用的三个选项的含义如下。

- -files  选项显示第一行信息，包含文件名称、文件大小、块数量和健康情况
- -blocks 选项描述文件中各个块的信息，每个块一行
- -racks 选项显示各个块的机架位置和datanode的地址

如果不指定任何参数，运行不带参数的hdfsfsck命令会显示完整的使用说明。

#### 3. datanode块扫描器

每个datanode运行一个块扫描器，定期检测本节点上的所有块，从而在客户端读取到坏块之前及时地检测和修复坏块。可以依靠扫描器所维护的块列表依次扫描块，查看是否有校验和错误，扫描器还使用节流机制，来维持datanode的磁盘带宽(即，块扫描器工作时仅占用一小部分磁盘带宽)。

在默认情况下，块扫描器每隔三周就会检测块，以应对可能的磁盘故障，该周期由dfs.datanode.scan.period.hours属性设置，默认值是504小时。损坏的块被报给namenode，并被及时修复。

用户可以访问datanode的网页（http://datanode:50075/blockScannerReport）来获取该datanode的块检测报告。以下是一份报告的范例，很容易理解：

```
Total Blocks : 21131
Verified in last hour : 70
Verified in last day : 1767
Verified in last week : 7360
Verified in last four weeks : 20057
Verified in SCAN PERIOO : 20057
Not yet verified : 1074
Verified since restart : 35912
Scans since restart : 6S41
Scan errors since restart : 0
Transient scan errors : 0
Current scan rate limit KBps : 1024
Progress this period : 109%
Time leftlin cur period ：53.08%
```

通过指定listblocks参数，http://datanode:50075/blockScannerReport?Listblocks会在报告中列出该datanode上所有的块及其最新验证状态。下面节选部分内容（由于页面宽度限制，报告中的每行内容被显示成两行）：

```
blk_6035596358209321442 : status : ok type : none scan time :
0 not yet verified
blk_3065580480714947643 : status : ok type : remote scan time :
1215755306400 2008-07-11 05:48:26,400
blk_8729669677359108508 : status:ok type:local scan time :
1215755727B45 2008-07-11 05:55:27,345
```

第一列是块ID，接下来是一些键-值对：

- 块的状态(status)：failed(损坏的)或ok(良好的)。由最近一次块扫描是否检测到校验和来决定。
- 扫描类型(type)：local(本地的)、remote(远程的）或none(没有)。如果扫描操作由后台线程执行，则是local；如果扫描操作由客户端或其他datanode执行，则是remote；如果针对该块的扫描尚未执行，则是none。
- 扫描时间：从1970年1月1号午夜开始到扫描时间为止的毫秒数，另外也提供更易读的形式。

#### 4. 均衡器

随着时间的推移，多个datanode上的块分布会越来越不均匀，不均衡的集群会降低MapReduce的本地性，导致部分datanode相对更加繁忙。应避免出现这种情况。

均衡器(balancer)程序是一个Hadoop守护进程，它将块从忙碌的datanode移到相对空闲的datanode，从而重新分配块。同时坚持块复本的放置策略，将复本分散到不同的机架，以降低数据损坏率(参见3.6.2节)。它不断移动块，直到集群到达均衡。即每个datanode的使用率(该节点上已使用的空间和空间容量之间的比率)和集群的使用率(集群中已使用的空间与集群的空间容量之间的比率)非常接近，差距不超过给定的阀值。可调用下面指令启动均衡器：

```sh
start-balancer.sh
```

-threshold参数指定阈值(百分比格式)，以判定集群是否均衡。该标记是可选的；若省略，默认阈值是10％。任何时刻，集群中都只运行一个均衡器。

均衡器会一直运行，直到集群变得均衡为止，此时，均衡器不能移动任何块，或失去对namenode的联络。均衡器在标准日志目录中创建一个日志文件，记录它所执行的每轮重新分配过程（每轮次输出一行）。以下是针对一个小集群的日志输出(为适应页面显示要求稍微调整了格式)：

```
Time Stamp                 Iteration Bytes Already MovedLeft  To Move.        Being Moved
Mar 18, 2009 5:23:42 PM 0                  0 KB                219.21 MB        150.29 MB
Mar 18, 2009 5:27:14 PM 1                  195.24MB            22.45 MB         150.29 MB
The cluster is balanced. Exiting...
Balancing took 6.072933B333B333B minutes 
```

为了降低集群负荷、避免干扰其他用户，均衡器被设计为在后台运行。在不同节点之间复制数据的带宽也是受限的。默认值是很小的1MB/s，可以通过方hdfs-site.xml文件中的`dfs.datanode.balance.bandwidthPerSec`属性重新设定(单位是字节)。

## 11.2 监控

监控是系统管理的重要内容。在本小节中，概述Hadoop的监控工具，看它们如何与外部监控系统相结合。

监控的目标在于检测集群在何时未提供所期望的服务。主守护进程是最需要监控的，包括主namenode、辅助namenode和资源管理器。可以预期少数datanode和节点管理器会出现故障，特别是在大型集群中。因此，需要为集群预留额外的容量，即使有一小部分节点宕机，也不会影响整个系统的运作。

除了以下即将介绍的工具之外，管理员还可以定期运行一些测试作业来检查集群的健康状况。

### 11.2.1 日志

所有Hadoop的守护进程都会产生日志文件，这些文件非常有助于查明系统中已发生的事件。10.3.2节在i寸论系统日志文件时解释了如何配置这些文件。

#### 1. 设置日志级别

在故障排查过程中，若能够临时变更特定组件的日志级别的话，将非常有益。

可以通过Hadoop守护进程(HDFS：50070，YARN：8088，seconday：50090，historyJob：19888)的网页(在守护进程的网页的/logs/level目录下)来改变任何logg4j日志名称的日志级别。一般来说，Hadoop中的日志名称对应着执行相关日志操作的类名称。此外，也有例外情况，因此最好从源代码中查找日志名称。

可以为所有给定前缀开始的类启用日志，例如，为了启用资源管理器相关的所有类的日志调试特性，可以访问网页http://resourcemanager:8088/loglevel，并将日志名`org.apache.hadoop.yarn.server.resourcemanager`设置为DEBUG级别。

也可以通过以下命令实现上述目标：

```sh
hadoop daemonlog -setlevel resource-manager-host:8088 \
org.apache.hadoop.yarn.server.resourcemanager DEBUG
```

按照上述方式修改的日志级别会在守护进程重启时被复位，通常这也符合用户预期。如果想永久性地变更日志级别，只需在配置目录下的log4j.properties件中添加如下这行代码：

```xml
log4j.logger.org.apache.hadoop.yarn.server.resourcemanager=DEBUG
```

#### 2. 获取堆栈跟踪

Hadoop守护进程提供一个网页（网页界面的/stacks目录）对正在守护进程的JVM中运行着的线程执行线程转储(thread dump)。例如，可通过http://resource-manager-host:8088/stacks获得资源管理器的线程转储。

### 11.2.2 度量和JMX(JAVA管理扩展)

Hadoop守护进程收集事件和度量相关的信息，这些信息统称为“度量”(metric)。例如，各个datanode会收集以下度量(还有更多)：写人的字节数、块的复本数和客户端发起的读操作请求数(包括本地的和远程的)。

------

有时用metrics2指代Hadoop2及后续版本的度量系统，以区别早期版本Hadoop的旧度量系统（现在已经不支持)。

------

度量从属于特定的上下文(context)。目前，Hadoop使用 **dfs**、**mapred**、**yarn**和 **rpc**四个上下文。Hadoop守护进程通常在多个上下文中收集度量。例如，datanode会分别为dfs和rpc上下文收集度量。

------

**度量和计数器的差别在哪里**

主要区别是应用范围不同：度量由Hadoop守护进程收集，而计数器(参见9.1节对计数器的讨论)先对MapReduce任务进行采集，再对整个作业进行汇总。此外，用户群也不同，从广义上讲，度量为管理员服务，而计数器主要为MapReduce用户服务。

二者的数据采集和聚集过程也不相同。计数器是mapReduce的特性，MapReduce系统确保计数器值由任务JVM产生，再传回application master，最终传回运行MapReduce作业的客户端。(计数器是通过RPC的心跳[heatbeat]传播的，详情可以参见7.1.5节)在整个过程中，任务进程和application master都会执行汇总操作。

度量的收集机制独立于接受更新的组件。有多种输出度量的方式，包括本地文件、Ganglia和JMX。守护进程收集度量，并在输出之前执行汇总操作。

------

所有的Hadoop度量都会分布给JMX(Java management Extensions)，可以使用标准的JMX工具，如JConsole(JDK自带)，查看这些度量。对于远程监控，必须将JMX系统属性com.sun.management.jmxremote.port(及其他一些用于安全的属性)设置为允许访问。为了在namenode实现这些，需要在hadoop-env.sh文件中设置以下语句：

```
HADOOP_NAMENODE_OPTS="-Dcom.sun.management.jmxremote.port=8004"
```

也可以通过特定Hadoop守护进程的/jmx网页查看该守护进程收集的JMX度量(JSON格式），这为调试带来了便利。例如，可以在网页http://namenode-host:50070/jmx查看namenode的度量。

Hadoop自带大量的度量通道用于向外部系统发布度量，例如本地文件或Ganglia监控系统。通道在hadoop-mertrics2.properties文件中配置，可以参考该文件，了解如何进行配置设置。

## 11.3 维护

### 11.3.1 日常管理过程

#### 1. 元数据备份

如果是namenode的永久性元数据丢失或损坏，整个文件系统无法使用。因此，元数据备份非常关键，可以在系统中分别保存若干份不同时间的备份(例如：1小时前、1天前、1周前或1个月前)，以保护元数据。方法一是直接保存这些元数据文件的复本；方法二是整合到namenode上正在使用的文件中。

最直接的元数据备份方法是使用dfsadmin命令下载namenode最新的fsimage文件的复本：

```
hdfs dfsadmin -fetchlmage fsimage.backup
```

可以写一个脚本从准备存储fsimage存档文件的异地站点运行该命令。该脚本还需测试复本的完整性。测试方法很简单，只要启动一个本地namenode守护进程，查看它是否能够将fsimage和edits文件载人内存（例如，通过扫描namenode日志以获得操作成功信息）。

#### 2. 数据备份

尽管HDFS已经充分考虑了如何可靠地存储数据，但是正如任何存储系统一样，仍旧无法避免数据丢失。因此，备份机制就很关键。Hadoop中存储着海量数据，判断哪些数据需要备份以及在哪里备份就极具挑战性。关键在于为数据划分不同的优先级，哪些无法重新生成的数据的优先级最高，这些数据对业务非常关键，同理，可再生数据和一次性数据商业价值有限，所以优先级最低，无需备份。

------

不要误以为HDFS的复本技术足以胜任数据备份任务，HDFS的程序纰漏、硬件故障都可能导致复本丢失。尽管Hadoop的设计方案可确保硬件故障极不可能导致数据丢失，但是这种可能性无法完全排除，特别是软件bug和人工误操作情况在所难兔。

再比较HDFS的备份技术和RAID。RAID可以确保在某一个RAID盘片发生故障时数据不受破坏，但是，如果发生RAID控制器故障、软件纰漏（可能重写部分数据）或整个磁盘阵列故障，数据肯定会丢失。

------

通常情况下，HDFS的用户目录还会附加若干策略，例如目录容量限制和夜间备份等。用户需要熟悉相关策略，才可以预料执行结果。

distcp时一个理想的备份工具，其必行的文件复制功能可以将备份文件存储到其他集群(最好软件版本不同，以防Hadoop文件纰漏而丢失数据)或其他Hadoop文件系统(例如S3)。此外，还可以用3.4节提到的方法将数据从HDFS导出到完全不同的存储系统中。

HDFS允许管理者和用户对文件系统进行快照。快照是对文件系统子树在给定时刻的一个只读复本。由于并不真正复制数据，因此快照非常高效，它们简单地记录每个文件的元数据和块列表，这对于重构快照时刻的文件系统内容已经足够了。

快照不是数据备份的替代品，但是对于恢复用户误删文件在特定时间点的数据而言，它们是一个有用的工具。可以制定一个周期性快照的策略，根据年份将快照保存一段特定的时间。例如，可以对前一天的数据进行每小时一次的快照，对前一个月的数据进行每天一次的快照。

#### 3. 文件系统检查（fsck）

建议定期地在整个文件系统上运行HDFS的凶文件系统检查）工具（例如，每天执行），主动查找丢失的或损坏的块。参见11.1.4节对文件系统检查的详细介绍。

#### 4. 文件系统均衡器

定期运行均衡器工具（参见11.1.4节对均衡器的详细介绍），保持文件系统的各个datanode比较均衡。

### 11.3.2 委任和解除节点

Hadoop集群的管理员经常需要向集群中添加节点，或从集群中移除节点。例如，为了扩大存储容量，需要委任节点。相反的，如果想要缩小集群规模，则需解除节点。如果某些节点表现反常，例如故障率过高或性能过于低下，则需要解除该节点。

通常情况下，节点同时运行datanode和节点管理器，因而两者一般同时被委任或解除。

#### 1. 委任新节点

新增一个节点非常简单：

- 配置hdfs-site.xml，指向namenode；
- 配置yarn-site.xml文件，指向资源管理器；
- 启动datanode和资源管理器守护进程。

然而，预先指定一些经过审核的节点以从中挑选新节点仍不失为一种好的方法。

随便允许一台机器以datanode身份连接到namenode是不安全的，因为该机器很可能会访问未授权的数据。此外，这种机器并非真正的datanode，不在集群的控制之下，随时可能停止，导致潜在的数据丢失。(想象一下，如果有多台这类机器连接到集群，而且某一个块的全部复本恰巧只存储在这类机器上，安全性如何？)由于错误配置的可能性，即使这些机器都在本机构的防火墙之内，这种做法的风险也很高。因此所有工作集群上的datanode以及节点管理器都应该被明确管理。

允许连接到namenode的所有datanode放在一个文件中，文件名由dfs.hosts属性指定，该文件放在的本地文件系统中，每行对应一个datanode的网络地址(由datanode报告——可以通过namenode的网页查看)。如果需要为一个datanode指定多个网络地址，可将多个网络地址放在一行，由空格隔开。类似的，可能连接到资源管理器的各个节点管理器也在同一个文件中指定（文件的名称由yarn.resourcemanager.nodes.include-path属性指定)。在通常情况下，**由于集群中的节点同时运行datanode和节点管理器守护进程，dfs.hosts和yarn.resourcemanager.nodes.include-path会同时指向一个文件，即include文件**。

------

dfs.hosts和yarn.resourcemanager.nodes.include-path属性指定的(一个或多个)文件不同于slaves文件。前者提供namenode和资源管理器使用，用于决定可以连接哪些工作节点。Hadoop控制脚本使用slaves文件执行面向整个集群范围的操作，例如重启集群等。Hadoop守护进程从不使用slaves文件。

------

向集群添加新节点的步骤如下：

1. 将新节点的网络地址添加到include文件中。

2. 运行以下命令，将审核通过的一系列datanode更新至namenode信息：

	```
	hdfs dfsadmin -refreshNodes
	```

3. 运行以下指令，将审核过的一系列节点管理器信息更新至资源管理器

	```
	yarn rmadmin -refreshNodes
	```

4. 以新节点更新slaves文件。这样的话，Hadoop控制脚本会将新节点包括在未来操作之中。

5. 启动新的datanode和节点管理器。

6. 检查新的和节点管理器是否都出现在网页界面中。

HDFS不会自动将块从旧的datanode移到新的datanode以平衡集群。用户需要自行运行均衡器，详情参考11.1.4节对均衡器的讨论。

#### 2. 解除旧节点

HDFS能够容忍datanode故障，但这并不意味着允许随意终止datanode。以三复本策略为例，如果同时关闭不同机架上的三个datanode，则数据丢失的概率会非常高。正确的方法是，用户将拟退出的若干datanode告知namenode，Hadoop系统就可在这些datanode停机之前将块复制倒其他datanode。

有个节点管理器支持，Hadoop对故障的容忍度更高，如果关闭一个正在运行的MapReduce任务的节点管理器，Application Master会检测到故障，并在其他节点上重新调度任务。

解除节点的过程由exclude文件控制，对于HDFS来说，文件由`dfs.hosts.exculde`属性设置；对于YARN来说，文件由`yarn.resourcemanager.nodes.exclude-path`属性设置，这些文件列出若干未被允许连接到集群的节点。通常，这两个属性指向同一个文件。

YARN判断一个node manager能否连接到资源管理器非常简单。仅当节点管理器出现在include文件且不出现在exclude文件中时，才能够连接到资源管理器。注意，如果未指定include文件，或include文件为空，则意味着所有节点都包含在include文件中。

HDFS的规则有所不同，如果一个datanode同时出现在include和exclude文件中，则该节点可以连接，但是很快会被解除委任。表11-3总结了datanode的不同组合方式。与节点管理器类似，如果未指定include文件或include文件为空，都意味着包含所有节点。

​								**表11-3 HDFS的include文件和exclude文件**

| 节点是否出现在include文件中 | 节点是否出现在exclude文件中 | 解释                 |
| --------------------------- | --------------------------- | -------------------- |
| 否                          | 否                          | 节点无法连接         |
| 否                          | 是                          | 节点无法连接         |
| 是                          | 否                          | 节点可连接           |
| 否                          | 否                          | 节点可连接，将被解除 |

从集群中移除节点的步骤如下：

1. 将待解除节点的网络地址添加到exclude文件中，不更新include文件；

2. 执行以下指令，使用一组新的审核过的来更新namenode设置

	```
	hdfs dfsadmin -refreshNodes
	```

3. 使用一组新的审核过的节点管理器来更新资源管理器设置：

	```
	yarn rmadmin -refreshNodes
	```

4. 转到网页界面，查看待解除datanode的管理状态是否已经变为“正在解除”(Decommission ln Progress)，因为此时相关的datanode正在被解除过程之中。这些datanode会把它们的块复制到其他datanode中。

5. 当所有datanode的状态变为“解除完毕"(Decommissioned)时，表明所有块都已经复制完毕。关闭已经解除的节点。

6. 从include文件中移除这些节点，并运行以下命令：

	```
	hdfs dfsadmin -refreshNodes 
	yarn rmadmin -refreshNodes
	```

7. 从slaves文件中移除节点

### 11.3.3 升级

升级Hadoop集群需要细致的规划，特别是HDFS的升级。如果文件系统的布局的版本发生变化，升级操作会自动将文件系统数据和元数据迁移到兼容新版本的格式。与其他涉及数据迁移的过程相似，升级操作暗藏数据丢失的风险，因此需要确保数据和元数据都已经备份完毕。参见11.3.1节对日常管理过程的讨论。

规划过程最好包括在一个小型测试集群上的测试过程，以评估是否能够承担（可能的）数据丢失的损失。测试过程使用户更加熟悉升级过程、了解如何配置本集群和工具集，从而为在生产集群上进行升级工作消除技术障碍。此外，一个测试集群也有助于测试客户端的升级过程。用户可以阅读以下补充内容中对客户端兼容性的讨论。

------

**兼容性**

将Hadoop升级到另外一个版本时，需要仔细考虑升级步骤，同时还需要考虑几个方面：API兼容性、数据兼容性和连接兼容性。

API兼容性重点考虑用户代码和发行的HadoopAPI之间的对比，例如JavaMapReduceAPI。主发行版本（例如从1.x.y到2.0.0）是允许破坏API兼容性的，因此，用户的程序要修改并重新编泽·次重点发行版本（例如从到1.0.x到1.1.0）和单点发行版本（例如从1.0.1到1.0.2）不应该破坏兼容性。

Hadoop针对API函数使用分类模式来表征其隐定性。按照先前的命名规则，API兼容性包括标记为lnterfacestability.Stable。公开发行的HadoopAPI中包含有部分函数，标记为InterfaceStability.Evolving或者InterfaceStability.Unstable(上述标注包含在org.apache.hadoop.classification软件包中），这意味允许它们分别在次重点发行版本和单点发行版本中破坏兼容性。

数据兼容性主要考虑持久化数据和元数据的格式，例如在HDFS namenode中用于存储持久数据的格式，这些格式允许在主版本和次重点版本之间修改，但是这类修改对用户透明，因为系统升级时数据会自动迁移。系统升级路径有一些限制，这些限制包含在发行须知中。例如，在系统升级过程中可能需要通过某个中间发行版本依次升级，而非一步直接升级到最新版本。

连接兼容性主要考虑通过利用RPC和HTTP这样的连接协议来实现客户端和服务器之间的互操作性。连接兼容性的规则是，客户端与服务器必须有相同的主版本号，但次版本号或单点发行版本号可以不同(例如，客户端2.0.2版可以和服务器2.0.1版或2.1.0版一起工作，但是与服务器3.0.0版不能一起工作)。

------

如果文件系统的布局并未改变，升级集群就非常容易：在集群上安装新版本的Hadoop（客户端也同步安装），关闭旧的守护进程，升级配置文件，启动新的守护进程，令客户端使用新的库。整个过程是可逆的，换言之，也可以方便地还原到旧版本。

成功升级版本之后，还需要执行两个清理步骤：

- 从集群中移除旧的安装和配置文件。
- 在代码和配置文件中针对“被弃用”(deprecation)警告信息进行修复。

升级功能是Hadoop集群管理工具如Cloudera Manager和Apache Ambari的一个亮点，它们简化了升级过程，且使得滚动升级变得容易。节点以批量方式升级(或对于主节点，一次升级一个)，这样客户端不会感受到服务中断。

#### HDFS的数据和元数据升级

如果采用前述方法来升级HDFS，且新旧HDFS的文件系统布局恰巧不同，则namenode无法正常工作，在其日志文件中产生如下信息：

```
File System image contains an old layout version -16
An upgrade to version -18 is required
Please restart NameNode with -upgrade option
```

最可靠的判定文件系统升级是否必要的方法是在一个测试集群做实验。

升级HDFS会保留前一版本的元数据和数据复本，但这并不意味着需要两倍的存储开销，因为datanode使用硬链接保存指向同一块的两个应用（分别为当前版本和前一版本），从而能够在需要时方便地回滚到前一版本。需要强调的是，系统回滚到旧版本之后，原先的升级改动都将被取消。

用户可以保留前一个版本的文件系统，但无法回滚多个版本。为了执行HDFS数据和元数据上的另一次升级任务，需要删除前一版本，该过程被称为“定妥升级”(finalizing the upgrade)。一旦执行该操作，就无法再回滚到前一个版本。

一般来说，升级过程可以忽略中间版本。但在某些情况下还是需要先升级到中间版本，这种情况会在发布说明文件中明确指出。

仅当文件系统健康时，才可升级，因此有必要在升级之前调用为工具全面检查文件系统的状态（参见11.1.4节对工具的讨论）。此外，最好保留不的输出报告，该报告列举了所有文件和块信息；在升级之后，再次运行新建一份输出报告并比较两份报告的内容。

在升级之前最好清空临时文件，包括HDFS的MapReduce系统目录和本地的临时文件等。

综上所述，如果升级集群会导致文件系统的布局变化，则需要采用下述步骤进行升级：

（1）在执行升级任务前，确保前一升级已经定妥。

（2）关闭YARN和MapReduce守护进程。

（3）关闭HDFS，并备份namenode目录。

（4）在集群和客户端安装新版本的Hadoop。

（5）使用 -upgrade选项启动HDFS。

（6）等待，直到升级完毕。

（7）校验HDFS是否运行正常。

（8）启动YARN和守护进程。

（9）回滚或定妥升级任务。

运行升级任务时，最好移除PATH环境变量下的Hadoop脚本，这样的话，用户就不会混淆针对不同版本的脚本。通常可以为新的安装目录定义两个坏境变量。在后续指令中，定义了OLDHADOOP_HOME和NEWHADOOP-HOME两个环境变量。

**启动升级** 	为了执行升级，可运行以下命令(即前述的步骤5)：

```sh
％ $NEW_HADOOP_HOME/bin/start-dfs.sh -upgrade
```

该命令的结果是让namenode升级元数据，将前一版本放在dfs.namenode.name.dir下的名为previous的新目录中。类似地，datanode升级存储目录，保留原先的复本，将其存放在previous目录中。

**等到，直到升级完成**	升级过程并非一蹴即就，可以用口查看升级进度，升级事件同时也出现在守护进程的日志文件中，步骤6：

```sh
％ $NEW_HADOOP_HOME/bin/hdfs dfsadmin -upgradeProgress status

upgrade for version -18 has been completed
Upgrade is not finalized
```

**检查升级情况**	显示升级完毕。在本阶段中，用户可以检查文件系统的状态，例如使用fsck(一个基本的文件操作）检验文件和块，参见步骤7。以防止其他用户修改数据。详见11.1.2节安全模式的有关内容。

**回滚升级(可选的)**	如果新版本无法正确工作，可以回滚到前一版本，参见步骤9，前提是尚未定妥更新。

------

回滚操作会将文件系统的状态转回到升级之前的状态，同期所做的任何改变都会丢失。换句话说，将回滚到文件系统的前一状态，而非将当前的文件系统降级到前一版本。

------

首先，关闭新的守护进程：

```bash
％ $NEW_HADOOP_HOME/bin/stop-dfs.sh
```

其次，使用-rollback选项启动旧版本的HDFS：

```bash
％ $OLD_HADOOP_HOME/bin/start-dfs.sh -rollback
```

该命令会让namenode和datanode使用升级前的复本替换当前的存储目录，文件系统返回之前的状态。

**定妥升级(可选)**	如果用户满意于新版本的HDFS，可以定妥升级，参见步骤9，以移除升级前的存储目录。

------

一旦升级定妥，就再也无法回滚到前一版本。

------

在执行新的升级任务之前，必须执行这一步：

```sh
％ $NEW_HADOOP_HOME/bin/hdfs dfsadmin -finalizeUpgrade

％ $NEW_HADOOP_HOME/bin/hdfs dfsadmin -upgradeProgress status

There are no upgrades inprogress.
```

现在，HDFS已经完全升级到新版本了。

