# HDFS Users Guide

## 1. Purpose

对于使用Hadoop分布式文件系统(HDFS)的用户来说，无论是作为Hadoop集群的一部分，还是作为独立的通用分布式文件系统，本文档都是一个起点。虽然HDFS被设计为在许多环境中“仅能工作”，但是有关HDFS的工作知识对于在特定集群上进行配置改进和诊断有很大帮助。

## 2. Overview

HDFS是Hadoop应用程序使用的主要分布式存储。HDFS集群主要由管理文件系统元数据的NameNode和存储实际数据的datanode组成。HDFS体系结构指南详细描述了HDFS。本用户指南主要处理使用HDFS集群的用户和管理员之间的交互。HDFS体系结构图描述了NameNode、datanode和客户机之间的基本交互。客户端为文件元数据或文件修改联系NameNode，并直接使用datanode执行实际的文件I/O。

以下是用户在集群搭建和优化中的一些特性：

- 包括HDFS在内的Hadoop非常适合使用普通硬件进行分布式存储和分布式处理。它是容错的、可伸缩的，并且扩展起来非常简单。MapReduce以其简单性和适用于大型分布式应用程序而闻名，它是Hadoop不可分割的一部分。
- HDFS是高度可配置的，其默认配置非常适合于许多安装。大多数时候，只需要针对非常大的集群调整配置。
- Hadoop是用Java编写的，所有主要平台都支持它。
- Hadoop支持与HDFS直接交互的类shell命令。
- NameNode和datanode已经内置在web服务器中，这使得检查集群的当前状态变得很容易。
- HDFS中经常实现新的特性和改进。以下是一些有用的功能:
	- 文件权限和身份验证
	- Rack awareness：在调度任务和分配存储时考虑节点的物理位置
	- Safemode：维护管理模式
	- fsck：用于诊断文件系统的健康状况、查找丢失的文件或块的实用程序
	- fetchdt: 获取委托令牌并将其存储在本地系统的文件中
	- Balancer：当数据不均匀地分布在各个数据节点时，用来平衡集群的工具
	- Upgrade and rollback：在软件升级之后，如果出现意外问题，可以回滚到升级之前的HDFS状态
	- Secondary NameNode：执行NameNode命名空间的定期检查点，并帮助将包含HDFS修改日志的文件大小保持在NameNode的特定限制内
	- Checkpoint node：执行命名空间的定期检查点，并帮助最小化存储在NameNode的日志的大小，其中包含对HDFS的更改。替换先前由次要名称节点填充的角色，但尚未进行强制替换。在NameNode没有Backup Node的情况下，NameNode允许同时有多个检查点节点。
	- Backup node：检查点节点的扩展。除了检查点之外，它还从NameNode接收编辑流，并在内存中维护自己的名称空间副本，该副本始终与活动的NameNode名称空间状态保持同步。一次只能用NameNode注册一个备份节点。

## 3. Prerequisites

以下教程说明了如何安装一个Hadoop集群：

- 初学者准备的[Single Node Setup](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/SingleCluster.html) 教程；
- 为大型分布式集群准备的[Cluster Setup](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/ClusterSetup.html)教程；

本文档的其余部分假设用户能够使用至少一个DataNode设置和运行HDFS。出于本文的目的，NameNode和DataNode可以在同一台物理机器上运行。

## 4. Web Interface

为了显示关于集群当前状态的基本信息，NameNode和DataNode都运行一个内部web服务器。使用默认配置，NameNode的首页位于http://namenode-name:9870/。它列出了集群中的数据节点和集群的基本统计信息。web界面还可以用来浏览文件系统(使用NameNode首页上的“浏览文件系统”链接)。

## 5. Shell Commands

Hadoop包含各种类似shell的命令，它们直接与HDFS和Hadoop支持的其他文件系统交互。命令库/hdfs dfs -help列出了Hadoop shell支持的命令。此外，命令箱/hdfs dfs -help命令名显示命令的更详细的帮助信息。这些命令支持大多数正常的文件系统操作，如复制文件、更改文件权限等。它还支持一些特定于HDFS的操作，比如更改文件的复制。有关更多信息，请参见 [File System Shell Guide](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/FileSystemShell.html).。

### 5.1 DFSAdmin Command

bin/hdfs dfsadmin命令支持一些与hdfs管理相关的操作。bin/hdfs dfsadmin -help命令列出了当前支持的所有命令。例如:

- **-report**：报告HDF的基本统计数据。其中一些信息也可以在NameNode首页上找到。
- **-safemode：**虽然通常不需要，但管理员可以手动进入或离开Safemode
- **-finalizeUpgrade：**删除上次升级时集群的备份，用于确定本次升级成功后操作
- **-refreshNodes：**更新接到namenode的节点。默认情况下，Namenode重新读取dfs.hosts文件中的datanode主机名。dfs.hosts.exclude定义要停用节点。若dfs.namenode.hosts.provider.classname设置为org.apache.hadoop.hdfs.server.blockmanagement.CombinedHostFileManager，则所有包含和排除主机都将在dfs.hosts定义的JSON文件中指定。
- **-printTopology：**打印集群的拓扑。显示一个由机架和datanode组成的树。

## 6. Secondary NameNode

NameNode将对文件系统的修改存储为附加到本机文件系统——editsLog。当NameNode启动时，它从镜像文件fsimage中读取HDFS状态，然后从editsLog文件读取元数据修改日志用于更新HDFS状态。然后，它将新的HDFS状态写入fsimage，并使用空的editsLog追加新的元数据操作。由于NameNode只在启动期间合并fsimage和editsLog，所以在繁忙的集群中，editsLog文件可能会随着时间的推移变得非常大。较大的editsLog的会导致下一次重新启动NameNode需要更长的时间。

Secondary NameNode定期合并fsimage和editsLog文件，并将editsLog文件大小限制在一定范围内。**它通常在与主NameNode不同的机器上运行，因为它的内存需求与主NameNode的相同**(重要)。

Secondary NameNode上检查点进程的启动由两个配置参数控制：

- dfs.namenode.checkpoint.period：默认设置为3600s，指定两个连续检查点之间的最大延迟，设置时间单位为秒；
- dfs.namenode.checkpoint.txns：定义了NameNode上未检查点事务的数量(即editsLog中的事物总数)，默认100W。在大于100万时，会将强制执行紧急检查点，即使没有达到检查点周期。

上述两个条件只要满足一个，都会在Secondary NameNode中执行fsimage和editsLog合并操作。

Secondary NameNode将最新的检查点存储在与主NameNode目录结构相同的目录中。因此，必要时，主NameNode总是准备好读取选中的图像。

## 7. Checkpoint Node

NameNode使用两个文件保存其命名空间：fsimage(名称空间的最新检查点)和editsLog(自检查点以来命名空间的更改日志)。当NameNode启动时，它合并fsimage和editsLog，以提供文件系统元数据的最新视图。然后NameNode用新的HDFS状态覆盖fsimage，并开启一个新的editsLog。

Checkpoint node**定期**创建命名空间的检查点，它从活动的NameNode下载fsimage和editsLog，在本地合并它们，并将镜像文件fsimage上传到活动的NameNode。Checkpoint node通常在与NameNode不同的机器上运行，因为它的内存需求与NameNode的相同。Checkpoint node由配置文件中指定节点上的bin/hdfs namenode-Checkpoint启动。

Checkpoint node或Backup node的位置及web接口是通过dfs.namenode.backup.address和dfs.namenode.backup.http-address配置的。

```xml
<property>
	<name>dfs.namenode.backup.address</name>
	<value> Myhost5:9002</value>
</property>
<property>
  <name>dfs.namenode.backup.http-address</name>
  <value> Myhost5:9003</value>
</property>
```

Checkpoint Node的检查点进程的开始由两个配置参数控制：

- dfs.namenode.checkpoint.period：默认设置为3600s，指定两个连续检查点之间的最大延迟，设置时间单位为秒；
- dfs.namenode.checkpoint.txns：定义了NameNode上未检查点事务的数量(即editsLog中的事物总数)，默认100W。在大于100万时，会将强制执行紧急检查点，即使没有达到检查点周期。

Checkpoint Node将最新的检查点存储在与NameNode目录结构相同的目录中。这允许在必要时，通过NameNode始终可以读取Checkpoint Nodefsimage。可以在集群配置文件中指定多个检查点节点。

## 8. Backup Node

BackupNode提供与checkpointNode相同的检查机制，并维护一个始终与活动NameNode状态同步的文件系统命名空间内存中最新副本。除了从NameNode接受文件系统的editsLog流并将其保存到磁盘之外，备份节点还将editsLog应用到内存中自己的名称空间副本中，从而创建命名空间的备份。即Backup Node内存中的命名空间=NameNode最新fsimage+NameNode的元数据修改日志随时修改。

backupNode不需要从活动的NameNode下载fsimage和editsLog来创建检查点，而checkpointNode与Secondary NameNode需要这样做，因为它已经在内存中存有HDFS命名空间的最新状态。Backup Node检查点进程更有效，因为它只需要将名称空间保存到本地fsimage文件并重置编辑。

由于backupNode在内存中维护命名空间的副本，其RAM需求与NameNode相同。

NameNode一次支持一个备份节点。如果使用备份节点，则不能注册任何检查点节点。将来会支持同时使用多个备份节点。

备份节点的配置方式与检查点节点相同。它由`bin/hdfs namenode -backup`启动。

备份节点或检查点节点的位置及其附带的web接口是通过dfs.namenode.backup.address和dfs.namenode.backup.http-address配置变量配置的。

使用备份节点提供了在没有持久存储的情况下运行NameNode的选项，从而将保存名称空间状态的所有责任委托给备份节点。为此，使用-importCheckpoint选项启动NameNode，同时为NameNode配置edits dfs.NameNode.edits.dir指定持久化存储目录。

有关创建备份节点和检查点节点背后的动机的完整讨论，请参见 [HADOOP-4539](https://issues.apache.org/jira/browse/HADOOP-4539)。有关命令的使用，请参阅namenode。

------

Secondary NameNode的作用现在（Hadoop2.x）可以被两个节点替换CheckpointNode和BackupNode。

## 9. Import Checkpoint

如果fsimage和editsLog的所有其他副本都丢失了，则可以将最新的检查点导入NameNode。操作步骤如下：

- 配置`dfs.namenode.name.dir`指定一个空目录
- 配置dfs.namenode.checkpoint.dir指定checkpointNode目录的位置；
- 使用`-importCheckpoint`选项启动NameNode。

NameNode将从dfs.namenode.checkpoint.dir目录获取检查点，然后将其保存到dfs.namenode.name.dir目录中，注意：该文件夹不能包含合法的fsimage，否则会执行失败。因为NameNode会检查fs.checkpoint.dir目录下镜像的一致性，但是不会做任何改动。

## 10. Balancer

HDFS数据可能并不总是均匀地放置在DataNode上。一个常见的原因是向现有集群添加新的datanode。在放置新块时，NameNode在选择存储新块的datanode之前会考虑各种参数：

- 将一个块的一个副本与写入该块的节点保持在同一节点上的策略
- 需要在机架间散布一个块的不同副本，这样集群才能在整个机架丢失后存活下来
- 其中一个副本通常与写入文件的节点放在同一个机架上，以减少跨机架网络I/O
- 将HDFS数据均匀地分布在集群中的各个datanode上

由于多种相互竞争的考虑，数据可能不会均匀地放置在各个datanode之间。HDFS为管理员提供了一个工具，用于分析整个DataNode上的块放置和重新平衡数据。Balancer管理员使用指南参考 [HADOOP-1652](https://issues.apache.org/jira/browse/HADOOP-1652)

 ## 11. Rack Awareness

HDFS集群可以识别放置每个节点的机架的拓扑结构。为了优化数据容量和使用，配置此拓扑非常重要。有关详细信息，请查看文档[rack awareness](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/RackAwareness.html) 。

## 12. Safemode

在启动期间，NameNode从fsimage和editsLog文件加载文件系统状态。然后，它等待datanode报告文件块信息，这样就不会过早地开始复制文件块，尽管集群中已经有足够的副本了。在这段时间内，NameNode处于Safemode。NameNode的Safemode本质上是HDFS集群的只读模式，它不允许对文件系统或块进行任何修改。通常，在DataNodes报告大多数文件系统块可用之后，NameNode会自动离开Safemode。如果需要，可以使用`bin/ HDFS dfsadmin -safemode`命令开启Safemode。NameNode的首页显示Safemode是开着还是关着。setSafeMode()的JavaDoc提供了更详细的描述和配置。

## 13. fsck

HDFS支持fsck命令来检查HDFS文件系统各种不一致性问题。它是为报告各种文件的问题而设计的，例如，文件缺少块或复制不足块。与用于本机文件系统的传统fsck不同，此命令不会更正它检测到的错误。通常，NameNode会自动更正大多数可恢复的故障。默认情况下，fsck会忽略打开的文件，但提供了一个在报告期间选择所有文件的选项。HDFS fsck命令不是Hadoop shell命令。它可以作为bin/hdfs fsck运行。有关命令的用法，请参见[fsck](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html#fsck).。fsck可以在整个文件系统上运行，也可以在文件的子集上运行。

## 14. fetchdt

HDFS支持fetchdt命令来获取委托令牌并将其存储在本地系统的文件中。这个令牌稍后可以用于从非安全客户机访问安全服务器(例如NameNode)。应用通过使用RPC或HTTPS(通过Kerberos)获取令牌，因此在运行之前需要Kerberos票据(运行kinit获取票据)。HDFS fetchdt命令不是Hadoop shell命令。它可以作为bin/hdfs的fetchdt DTfile运行。获得令牌后，可以通过将HADOOP_TOKEN_FILE_LOCATION环境变量指向委托令牌文件来运行HDFS命令，而不需要Kerberos票据。有关命令的使用，请参阅[fetchdt](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html#fetchdt)命令。

## 15. Recovery Mode

通常，集群会配置多个元数据存储位置。如果一个存储位置损坏，可以从另一个存储位置读取元数据。但是，如果惟一可用的存储位置损坏了，该怎么办呢？在这种情况下，有一种特殊的NameNode启动模式称为恢复模式，它允许恢复大部分数据。

可以使用NameNode: NameNode - recovery启动NameNode恢复模式。在恢复模式下，NameNode将在命令行交互式地提示可以采取哪些操作来恢复数据。因为恢复模式会导致数据丢失，所以在使用编辑日志和fsimage之前，应该始终备份它们。

## 16. Upgrade and Rollback

与任何软件升级一样，当Hadoop在现有集群上升级时，可能会出现影响现有应用程序的新bug或不兼容的更改，而这些bug或更改之前没有发现。在任何重要的HDFS安装中，都不能丢失任何数据，更不要说从头重新启动HDFS了。HDFS允许管理员会退到早期版本的Hadoop，并将集群回滚到升级之前的状态。HDFS升级在 [Hadoop Upgrade](http://wiki.apache.org/hadoop/Hadoop_Upgrade)有更详细的描述。HDFS一次可以有一个这样的备份。升级完成后，管理员需要使用bin/hadoop dfsadmin -finalizeUpgrade命令删除现有备份。下面简要介绍一下典型的升级过程:

- 在升级Hadoop软件之前，确定是否有现有的备份
- 停止集群并分发Hadoop的新版本
- 运行带有-upgrade选项的新版本(sbin/start-dfs.sh -upgrade)
- 大多数时候，集群都可以正常工作。一旦认为新的HDFS可以正常工作(可能在几天之后)，就完成升级。注意，在集群完成之前，删除升级之前存在的文件不会释放datanode上的实际磁盘空间。
- 如果需要回到旧版本：
	- 停止集群并分发Hadoop的早期版本
	- 在namenode上运行rollback命令(bin/hdfs namenode -rollback)
	- 使用rollback选项启动集群。(sbin / start-dfs.sh -rollback)

升级到新版本的HDFS时，需要重命名或删除新版本HDFS中的保留路径(为设置保留的存储空间)。如果NameNode在升级期间遇到保留路径，它将打印如下错误：

```
/.reserved is a reserved path and .snapshot is a reserved path component in this version of HDFS. Please rollback and delete or rename this path, or upgrade with the -renameReserved [key-value pairs] option to automatically rename these paths during upgrade.
```

指定`-upgrade -renameReserved`[可选键-值对]，可以在NameNode启动期间找到的所有保留路径，并自动重命名。例如，要将所有名为.snapshot的路径重命名为.my-snapshot，并将.reserved重命名为.my-reserved，用户可以指定-upgrade -renameReserved .snapshot=.my-snapshot，.reserved=.my-reserved。

如果没有使用-renameReserved指定键值对，NameNode将使用`.<LAYOUT-VERSION>.UPGRADE_RENAMED`作为保留路径的后缀。例如：`snapshot`更新为`snapshot.-51.UPGRADE_RENAMED`。

**注意：**如果可能，建议在升级之前先使用hdfs dfsadmin -saveNamespace。这是因为，如果编辑日志中操作会修改自动重命名文件，则可能导致数据不一致。

## 17. DataNode Hot Swap Drive

Datanode支持热插拔驱动器。用户可以添加或替换HDFS数据卷，而无需关闭DataNode。下面简要介绍一下典型的热交换驱动程序:

- 用户应该对它们进行格式化并适当地装载
- 
- 用户执行`dfsadmin -reconfig datanode HOST:PORT `来启动重新配置过程，可以使用`dfsadmin - refig datanode HOST:PORT status`来查询重新配置任务的运行状态。
- 重新配置任务完成后，用户可以安全地挂载删除的数据卷目录并物理删除磁盘。

## 18. File Permissions and Security

文件权限设计为类似于Linux等其他常见平台上的文件权限。目前，安全性仅限于简单的文件权限。启动NameNode的用户被视为HDFS的超级用户。HDFS的未来版本将支持诸如Kerberos之类的网络身份验证协议，用于用户身份验证和数据传输的加密。详细信息在权限指南中讨论。

## 19. Scalability

Hadoop目前运行在具有数千个节点的集群上。 [PoweredBy](http://wiki.apache.org/hadoop/PoweredBy)Wiki页面列出了一些在大型集群上部署Hadoop的组织。对于每个集群，HDFS都有一个NameNode。目前，NameNode上可用的总内存是主要的可伸缩性限制。在非常大的集群上，增加存储在HDFS中的文件的平均大小有助于在不增加NameNode内存需求的情况下增加集群大小。默认配置可能不适合非常大的集群。 [FAQ](http://wiki.apache.org/hadoop/FAQ)Wiki页面列表建议对大型Hadoop集群进行配置改进。

## 20. Related Documentation

- [Hadoop Site](http://hadoop.apache.org/) Apache Hadoop站点的主页
- [Hadoop Wiki](http://wiki.apache.org/hadoop/FrontPage) Hadoop Wiki的主页(FrontPage)。不像发布的文档是Hadoop源代码树的一部分，Hadoop Wiki是由Hadoop社区定期编辑的
- [FAQ](http://wiki.apache.org/hadoop/FAQ): FAQ Wiki页面。
- [Hadoop JavaDoc API](https://hadoop.apache.org/docs/stable/api/index.html)
- [hdfs-default.xml](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml)。它包括对大多数可用配置变量的简要描述
- [HDFS Commands Guide](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html): HDFS命令用法。

