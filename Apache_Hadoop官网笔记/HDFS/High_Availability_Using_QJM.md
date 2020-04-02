# HDFS High Availability Using the Quorum Journal Manager

## 1. 文档概述

本指南概述了HDFS高可用性(HA)特性，以及如何使用Quorum Journal Manager配置和管理高可用的HDFS集群。

本文档假设读者对HDFS集群中的组件和节点类型已经有了大致了解，有关详细信息，请参阅HDFS_Architecture。

## 2. 使用Quorum Journal Manager或常规共享存储

本指南讨论如何使用QJM配置和使用HDFS HA，以便在活动和备用namenode之间共享编辑日志。本指南旨在如何使用Observer NameNode配置HDFS HA。如何使用NFS而不是QJM为共享存储配置HDFS HA的信息，请参阅本 [this alternative guide.](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HDFSHighAvailabilityWithNFS.html)。

## 3. 背景

在Hadoop 2.0.0之前，NameNode是HDFS集群中的单点故障(SPOF)。每个群集都有一个NameNode，如果该机器或进程不可用，则整个群集将不可用，直到NameNode重新启动或在另一台机器上重新部署。

单NameNode以下两个主要方面影响了HDFS群集总体的可用性：

- 在意外事件(例如如机器崩溃)的情况下，集群将不可用，直到管理员重新启动NameNode。
- 机器日常维护(如NameNode计算机上的软件或硬件升级)，将导致群集停机时间窗口。

HDFS的高可用性特性是通过同一集群中运行两个(从3.0.0开始超过两个)冗余NameNodes的选项来解决上述问题，NameNode采用主/从配置，并具有热备用。这允许在NameNode节点机器崩溃的情况下快速故障转移到新的NameNode，或者为了日常维护而由管理员启动的正常故障转移。

## 4. HA集群结构

在典型的HA集群中，有两台或多台独立的NameNodes节点。在任何时间点，都有一个NameNode处于活动状态，其他的处于备用状态。活动的NameNode负责集群中的所有客户端操作，而备用节点只是充当worker，保持集群状态的更新，以便在必要时提供快速故障转移。

为了保证备用节点与活动节点集群状态的同步，两个节点都与一组名为**JournalNodes(JNS)**的独立守护进程通信。当活动节点执行任何HDFS命名空间修改时，会将修改记录同步发送给JNS进程，大部分JournalNodes持久化修改记录。备用NameNode节点从JNS读取编辑日志，并变化应用于自己的命名空间。在发生故障转移的情况下，备用NameNode将在转换为活动状态前，同步JournalNodes中所有编辑日志，以确保在发生故障转移之前完全同步主NameNode命名空间状态。

为了提供快速故障转移，还需要Standby NameNode节点具有集群所有文件块位置的最新信息。为了实现这一点，DataNode被配置所有NameNode的位置，并向所有NameNode节点发送块位置信息和心跳信号。

一次只有一个NameNode处于活动状态对于HA集群的正确操作至关重要。否则，名称空间状态将很快在两者之间出现差异，从而可能导致数据丢失或其他不正确的结果(这种情况称为**脑裂**)。为了确保数据正确并防止脑裂发生，JournalNodes一次只允许一个NameNode作为writer。在故障转移期间，要变为活动的NameNode将很快地接管写入JournalNodes的角色，这将有效地防止另一个NameNode继续处于活动状态，从而安全地继续进行故障转移。

## 5. 硬件资源

为了部署HA集群，需要准备以下设备：

- **NameNode machines**  运行活动和备用NameNode节点的机器应该具有相同的硬件设施，每台NameNode节点机器配置与非HA集群中使用的硬件一致。
- **JournalNode machines**   运行JournalNodes的机器。JournalNode守护程序相对较轻量级，因此这些守护程序可以合理地与其他Hadoop守护程序(例如NameNodes、YARN ResourceManager)配置在同一机器上。注意：必须至少有3个JournalNode守护进程，因为编辑日志修改必须写入大多数JNS，在单个JournalNode失效情况下，集群会正常运行。也可以运行3个以上的journalnode，但是为了实际增加系统能够容忍的故障数，应该运行奇数个JournalNode(即3、5、7等)。请注意，当使用N个JournalNode运行时，系统最多可以容忍$(N-1)/2$个故障并继续正常工作。

注意：在HA集群中，备用NameNodes负责命名空间状态的检查点机制，因此无需在HA集群中同时运行SecondaryNameNode、CheckpointNode或BackupNode。实际上如果你运行这些服务的话会出错。

## 6. 部署

### 6.1 配置概述

与Federation configuration类似，HA configuration会向后兼容(**backward compatible**) 并且已经存在的单个NameNode继续工作，不用更改任何配置。新配置的设计使得集群中的所有节点可能具有相同的配置，而无需根据节点的类型将不同的配置文件部署到不同的节点。

与HDFS Federation相同，HA集群也用**nameservice ID**来标识实际上可能由多个NameNodes组成的单个HDFS实例。此外，NameNode ID的概念被引入到HA集群中。群集中每个不同的NameNode都有不同的NameNode ID作为区分。为了支持所有NameNode使用单个配置文件，每一个 Namenode和Secondary Namenode/BackupNode/Checkpointer 加上以自己对应的NameServiceID 为后缀的参数.

### 6.2 配置详情

要配置HA NameNodes，必须将几个配置选项添加到hdfs-site.xml配置文件中。

这些配置在配置文件中的顺序并不重要，但是为`dfs.nameservices`和`dfs.ha.namenodes.[nameserviceID]`选择的值将决定后面这些配置的关键，应该在设置其余配置选项之前设定这些值。

- **dfs.nameservices**  nameservice的逻辑名称，例如“mycluster”。它将同时用于配置和集群中HDFS绝对路径的权限组件。

	**注意：**如果还使用HDFS Federation配置，则此配置还应包括其他nameservice列表，之间使用逗号分隔。

	```xml
	<property>
	  <name>dfs.nameservices</name>
	  <value>mycluster</value>
	</property>
	```

- **dfs.ha.namenodes.[nameservice ID]**   nameservices中每个namenode的唯一标识符。使用逗号分隔的NameNode id列表进行配置，用于DataNodes确定集群中的所有NameNode。例如，如果以前使用“mycluster”作为nameservice ID，并且希望使用“nn1”、“nn2”和“nn3”作为NameNode的ID，则可以这样配置：

	```xml
	<property>
	  <name>dfs.ha.namenodes.mycluster</name>
	  <value>nn1,nn2, nn3</value>
	</property>
	```

	注意：HA的NameNodes的最小数目是2，但是可以配置更多。由于通信开销，建议不超过5个(推荐3个NameNode）。

- **dfs.namenode.rpc-address.[nameservice ID].[name node ID]**   RPC通信地址与端口

	```xml
	<property>
	  <name>dfs.namenode.rpc-address.mycluster.nn1</name>
	  <value>machine1.example.com:8020</value>
	</property>
	<property>
	  <name>dfs.namenode.rpc-address.mycluster.nn2</name>
	  <value>machine2.example.com:8020</value>
	</property>
	<property>
	  <name>dfs.namenode.rpc-address.mycluster.nn3</name>
	  <value>machine3.example.com:8020</value>
	</property>
	```

	注意：也可以类似地配置“servicerpc address”设置。

- **dfs.namenode.http-address.[nameservice ID].[name node ID]**  HTTP协议端口

	```
	<property>
	  <name>dfs.namenode.http-address.mycluster.nn1</name>
	  <value>machine1.example.com:9870</value>
	</property>
	<property>
	  <name>dfs.namenode.http-address.mycluster.nn2</name>
	  <value>machine2.example.com:9870</value>
	</property>
	<property>
	  <name>dfs.namenode.http-address.mycluster.nn3</name>
	  <value>machine3.example.com:9870</value>
	</property>
	```

	注意：如果启用了Hadoop的安全功能，那么还应该为每个NameNode设置https地址。

- **fs.namenode.shared.edits.dir**  标识NameNodes将写入/读取编辑日志的JNS组URI。可以配置共享编辑日志存储的JournalNodes的地址，由活动nameNode写入并由备用nameNode读取，以便与活动nameNode文件系统状态保持最新。尽管必须指定多个JournalNode地址，但应该只配置其中一个uri。URI的格式应为：qjournal://*host1:port1*；*host2:port2*；*host3:port3*/*journalId*。

	journalId是nameservice的唯一标识符，它允许一组JournalNodes为多个nameservice提供存储。例如，如果此群集的JournalNodes在node1.example.com、node2.example.com和node3.example.com机器上运行，并且nameservice ID为mycluster，则可以使用以下值设置(JournalNode的默认端口为8485)：

	```xml
	<property>
	  <name>dfs.namenode.shared.edits.dir</name>		       <value>qjournal://node1.example.com:8485;node2.example.com:8485;node3.example.com:8485/mycluster</value>
	</property>
	```

- **dfs.client.failover.proxy.provider.[nameservice ID]**  HDFS客户端联系活动NameNode的Java类。当前Hadoop提供的两个实现ConfiguredFailoverProxyProvider和RequestHedgingProxyProvider。ConfiguredFailoverProxyProvider同时调用所有namenodes来确定活动的namenode，在随后的请求中，调用活动的namenode，直到发生故障转移。因此，除非使用自定义代理提供程序，否则请使用其中一个，例如：

	```xml
	<property>
	  <name>dfs.client.failover.proxy.provider.mycluster</name>
	 <value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
	</property>
	```

- **dfs.ha.fencing.methods**  在故障转移期间用于隔离活动NameNode的脚本或Java类的列表。

	为了保证系统的正确性，在给定的时间内只有一个NameNode处于活动状态。重要的是，当使Quorum Journal Manager用时，只允许一个NameNode写入JournalNode，因此不存在脑裂情况损坏文件系统元数据的可能性。

	故障转移期间使用的隔离方法配置为回车分隔列表，将按顺序尝试该列表，直到提示隔离已成功为止。Hadoop附带两种方法：shell和sshfence：

	- **sshfence**：SSH到活动NameNode并终止进程；sshfence选项将切换到目标节点，并使用fuser终止侦听服务的进程。需要必须能够在不提供密码短语的情况下通过SSH连接到目标节点。因此，还必须配置dfs.ha.fencing.ssh.private-key-files选项，该选项是ssh私钥文件的逗号分隔列表。例如：

		```xml
		<property>
			<name>dfs.ha.fencing.methods</name>
			<value>sshfence</value>
		</property>
		
		<property>
			<name>dfs.ha.fencing.ssh.private-key-files</name>
			<value>/home/exampleuser/.ssh/id_rsa</value>
		</property>
		```

	- **shell：**运行任意shell命令来隔离活动的NameNode

		```xml
		<property>
			<name>dfs.ha.fencing.methods</name>
			<value>shell(/path/to/my/script.sh arg1 arg2 ...)</value>
		</property>
		```

- **fs.defaultFS**  Hadoop文件系统客户端在未给定任何路径前缀时，使用的默认路径前缀。如果之前使用“mycluster”作为nameservice ID，这将是所有HDFS路径的authority部分的值。在core-site.xml文件中可以这样配置：

	```xml
	<property>
	  <name>fs.defaultFS</name>
	  <value>hdfs://mycluster</value>
	</property>
	```

- **dfs.journalnode.edits.dir**   JournalNode守护进程存储其本地状态的路径。这是JournalNode机器上的绝对路径，JNS使用的编辑和其他本地状态将存储在其中。此配置只能使用一个路径。通过运行多个单独的journalnode，或在本地连接的RAID阵列上配置此目录，可以提供此数据的冗余。例如：

	```xml
	<property>
	  <name>dfs.journalnode.edits.dir</name>
	  <value>/path/to/journal/node/local/data</value>
	</property>
	```

### 7. 部署详情

设置完所有必需的配置选项后，必须在集群上上启动JournalNode守护程序。可以通过运行命令`hdfs--daemon start journalnode`并等待daemon在每个相关机器上启动来完成。

启动JournalNodes后，必须首先同步两个HA NameNodes的磁盘元数据：

- 如果要设置一个新的HDFS集群，应该首先在其中一个NameNodes上运行format命令：HDFS namenode-format
- 如果已经格式化了NameNode，或者正在将未启用HA的集群转换为启用HA，那么现在应该通过在未格式化的NameNode上运行命令`hdfs NameNode-bootstrapStandby`将NameNode元数据目录的内容复制到另一个未格式化的NameNode。运行此命令还将确保JournalNodes(由dfs.namenode.shared.edits.dir配置)包含足够的编辑事务，以便能够启动两个NameNodes。
- 如果要将非HA NameNode转换为HA，则应运行命令`hdfs NameNode-initializeSharedEdits`，该命令将使用本地NameNode编辑目录中的编辑数据初始化JournalNodes。

此时，可以启动所有HA NameNodes，就像通常启动NameNode一样。

通过浏览每个NameNode的配置HTTP地址，可以分别访问它们的web页面。应该注意到，在配置的地址旁边将是NameNode的HA状态(standby或active）每当HA NameNode启动时，它最初都处于standby。

## 8. 管理员命令

现在已经配置并启动了HA NameNodes，将可以访问一些其他命令来管理HA HDFS集群。具体来说，应该熟悉`hdfs haadmin`命令的所有子命令。在没有任何附加参数的情况下运行此命令将显示以下用法信息：

```
Usage: haadmin
    [-transitionToActive <serviceId>]
    [-transitionToStandby <serviceId>]
    [-failover [--forcefence] [--forceactive] <serviceId> <serviceId>]
    [-getServiceState <serviceId>]
    [-getAllServiceState]
    [-checkHealth <serviceId>]
    [-help <command>]
```

- **transitionToActive** and **transitionToStandby** 将制定nameNode状态转换为active/standby状态；

	这些子命令分别使给定的NameNode转换为Active或Standby状态。这些命令不试图执行任何阻拦，因此很少使用。相反，更喜欢使用hdfs haadmin-failover子命令。

- **failover**  在两个NameNode之间启动故障转移

	此子命令将开启从第一个NameNode到第二个NameNode的故障转移。如果第一个NameNode处于待机状态，则此命令只需将第二个NameNode转换为活动状态而不会出错。如果第一个NameNode处于活动状态，则会尝试将正常转换为待机。如果失败，将按顺序尝试隔离方法(由dfs.ha.fencing.methods配置)，直到成功为止。只有在此进程之后，第二个NameNode才会转换为活动状态。如果没有fencing method成功，则第二个NameNode将不会转换为活动状态，并返回错误。

- **getServiceState**  确定给定的NameNode是活动的还是备用的

- **getAllServiceState** 获取所有namenode状态信息

- **checkHealth**  检查给定namenode健康情况。连接到指定NameNode以检查其运行状况。NameNode能够对自身执行一些诊断，包括检查内部服务是否按预期运行。如果NameNode正常，此命令将返回0，否则返回非0。可以使用此命令进行监视。

## 9. 跟踪正在进行的编辑日志

在默认设置下，secondaryNameNode将仅应用已完成的编辑日志段。如果希望SecondaryNameNode具有最新的名称空间信息，则可以启用跟踪正在进行的日志编辑段。此设置将尝试从JournalNodes上的内存缓存中获取编辑，并可以将事务应用于SecondaryNameNode的延迟时间减少到毫秒级别。如果无法从缓存中提供编辑服务，则standby状态仍将能够检索该编辑，但延迟时间将更长。相关配置如下：

- **dfs.ha.tail-edits.in-progress**  是否启用跟踪正在进行的编辑日志。这也将在JournalNodes上启用内存中的日志编辑缓存。默认情况下禁用。
- **dfs.journalnode.edit-cache-size.bytes** JournalNode上编辑日志的内存缓存大小。在典型的环境中，每个编辑日志大约需要200字节。例如，默认值1048576（1MB）可以容纳大约5000个事务。

## 10. 自动故障转移

### 10.1 简介

以上部分描述如何配置手动故障转移。在该模式下，即使活动节点发生故障，系统也不会自动触发从活动节点到secondaryNamenode的故障转移。本节介绍如何配置和部署自动故障转移。

### 10.2 组件

自动故障转移为HDFS部署添加了两个新组件：ZooKeeper和ZKFailoverController进程(缩写为ZKFC)。

Apache ZooKeeper是一个高度可用的服务，用于维护少量的协调数据、通知客户端数据的更改以及监视机器故障。自动HDFS故障转移的实现依赖于ZooKeeper完成以下任务：

- **故障检测**  群集中的每个NameNode节点都在ZooKeeper中维护一个持久session。如果节点机器崩溃，ZooKeeper会话将过期，从而通知其他NameNode应触发故障转移。
- **活跃NameNode选举**  ZooKeeper提供了一个简单的机制来独占地选择一个节点为Active。如果当前活动的NameNode崩溃，另一个节点可能会在ZooKeeper中使用一个特殊的独占锁，指示它应该成为下一个活动节点。

ZKFailoverController(ZKFC)是一个Hadoop新组件，它是一个ZooKeeper客户端，用于监视和管理NameNode的状态。运行NameNode的每台机器也运行一个ZKFC，ZKFC负责：

- **运行状况监视**  ZKFC使用运行状况检查命令定期ping其本地NameNode。只要NameNode及时响应，ZKFC就认为该节点健康。如果节点已崩溃、冻结或其他方式进入不正常状态，ZKFailoverController会将其标记为不正常。
- **ZooKeeper会话管理**  当本地NameNode正常时，ZKFC在ZooKeeper中保持会话打开。如果本地NameNode处于活动状态，它同时也持有一个“排他锁”znode；如果会话过期，锁节点将被自动删除。
- **ZooKeeper选举机制**  如果本地NameNode是健康的，并且ZKFC看到当前没有其他节点持有锁znode，那么它自己将尝试获取锁。如果成功，则它“赢得了选举”，并负责运行故障转移以使其本地NameNode处于活动状态。故障转移过程类似于上面描述的手动故障转移：首先，如果需要，前一个active节点被隔离，然后本地NameNode转换为active状态。

### 10.3 部署Zookeeper

在典型的部署中，ZooKeeper守护进程配置为在三个或五个节点上运行。由于ZooKeeper本身具有轻资源需求，因此可以将ZooKeeper节点与HDFS NameNode和SecondaryNameNode配置在同一硬件上。许多管理员选择将第三个ZooKeeper进程部署在与YARN ResourceManager相同的节点上。但是，建议将ZooKeeper节点配置为将其数据与HDFS元数据存储在不同的磁盘驱动器上，以获得最佳性能和隔离。

### 10.4 开始配置前

在开始配置自动故障转移之前，应关闭群集。群集运行时，无法从手动故障转移设置过渡到自动故障转移设置。

### 10.5 配置自动故障转移

自动故障转移的配置需要在配置文件中添加两个配置。在hdfs-site.xml文件中，添加：

```xml
<property>
  <name>dfs.ha.automatic-failover.enabled</name>
  <value>true</value>
</property>
 
<property>
  <name>ha.zookeeper.quorum</name>
  <value>zk1.example.com:2181,zk2.example.com:2181,zk3.example.com:2181</value>
</property>
```

列出运行ZooKeeper服务的主机和端口。

与文档前面描述的参数一样，这些设置可以在每个名称服务的基础上配置，方法是用名称服务ID作为配置键的后缀。例如，在启用联合的集群中，通过设置dfs.ha.automatic-failover.enabled.my-nameservice-id，只能显式地为其中一个nameservices启用自动故障转移。

### 10.6 在Zookeeper中初始化HA状态

在ZooKeeper中初始化所需的状态。可以通过从一个NameNode主机运行以下命令来执行此操作。

```bash
[hdfs]$ $HADOOP_HOME/bin/hdfs zkfc -formatZK
```

这将在ZooKeeper中创建一个znode，自动故障转移器将在其中存储其数据。

### 10.7 启动HDFS

由于在配置中启用了自动故障转移，start-dfs.sh脚本现在将在运行NameNode的任何计算机上自动启动ZKFC守护进程。当ZKFCs启动时，它们将自动选择一个要激活的NameNode。

### 10.8 手动启动群集

如果运行的是安全群集，则可能需要确保ZooKeeper中存储的信息也是安全的。这可防止恶意客户端修改ZooKeeper中的元数据或可能触发错误故障转移。

为了保护ZooKeeper中的信息，请首先将以下内容添加到core-site.xml文件中：

```xml
<property>
   <name>ha.zookeeper.auth</name>
   <value>@/path/to/zk-auth.txt</value>
 </property>
 <property>
   <name>ha.zookeeper.acl</name>
   <value>@/path/to/zk-acl.txt</value>
 </property>
```

注意：上述配置值中的“@”字符是指定配置不是inline的，而是指向磁盘上的文件。认证信息也可以通过CredentialProvider读取(请参阅hadoop common项目中的CredentialProviderAPI指南)。

第一个配置的文件指定ZooKeeper身份验证的列表，格式与ZK CLI使用的格式相同。例如，可以指定如下内容：

```
digest:hdfs-zkfcs:mypassword
```

其中hdfs zkfcs是ZooKeeper的唯一用户名，mypassword是用作密码的唯一字符串。

接下来，使用如下命令生成与此身份验证对应的ZooKeeper ACL：

```sh
[hdfs]$ java -cp $ZK_HOME/lib/*:$ZK_HOME/zookeeper-3.4.2.jar org.apache.zookeeper.server.auth.DigestAuthenticationProvider hdfs-zkfcs:mypassword

output: hdfs-zkfcs:mypassword->hdfs-zkfcs:P/OQvnYyU/nF/mGYvB/xurX8dYs=
```

面复制并粘贴此输出的部分到文件zk-acls.txt中，前缀为字符串“digest:”。例如：

```
digest:hdfs-zkfcs:vlUvLnd8MlacsE80rDuu6ONESbM=:rwcda
```

为了使这些acl生效，按照上面的描述重新运行zkfc-formatZK命令。

执行此操作后，可以按如下方式从ZK CLI验证acl：

```
[zk: localhost:2181(CONNECTED) 1] getAcl /hadoop-ha
'digest,'hdfs-zkfcs:vlUvLnd8MlacsE80rDuu6ONESbM=
: cdrwa
```

### 10.9 验证自动故障转移

设置自动故障转移后，应测试其操作。为此，首先找到活动的NameNode。通过访问NameNode web界面来判断哪个节点处于活动状态，每个节点在页面顶部报告其HA状态。

找到活动的NameNode后，可能会导致该节点出现故障。例如，可以使用kill-9< pid of NN>来模拟JVM崩溃。或者，您可以重启机器或拔下其网络接口以模拟不同类型的中断。触发要测试的中断后，另一个NameNode应在几秒钟内自动变为活动状态。检测故障并触发故障转移所需的时间取决于ha.zookeeper.session-timeout.ms的配置，但默认为5秒。

如果测试不成功，则可能是配置错误。检查zkfc守护进程和NameNode守护进程的日志，以便进一步诊断问题。

## 11. 自动故障转移FAQ

1. 以任何特定顺序启动ZKFC和NameNode守护进程是否重要？

	不。在任何节点上，可以在相应的NameNode启动之前或之后启动ZKFC。

2. 应该进行哪些额外的监控

	应该在运行NameNode的每个主机上添加监视，以确保ZKFC保持运行。例如，在某些类型的ZooKeeper故障中，ZKFC可能意外退出，应重新启动以确保系统准备好自动故障转移。

	此外，应该监视ZooKeeper仲裁中的每个服务器。如果ZooKeeper崩溃，则自动故障转移将不起作用。

3. 如果Zookeeper集群崩溃怎么办？

	如果ZooKeeper集群崩溃，则不会触发自动故障转移。不过，HDFS将继续运行，不会产生任何影响。当ZooKeeper重新启动时，HDFS将无问题地重新连接。

4. 可以指定一个NameNode为Active状态首选吗？

	不，目前不支持。首先启动的NameNode将变为活动的。可以选择按特定顺序启动群集，以便首选节点首先启动。

5. 配置自动故障转移时，如何启动手动故障转移？

	即使配置了自动故障转移，也可以使用相同的hdfs haadmin命令启动手动故障转移。它将执行协调的故障转移。

## 12. 启用HA的HDFS升级/终结/回滚

在HDFS版本之间移动时，有时只需安装较新的软件并重新启动集群即可。但是，有时升级正在运行的HDFS版本可能需要更改磁盘上的数据。在这种情况下，必须在安装新软件后使用HDFS Upgrade/Finalize/Rollback工具。在HA环境中，这个过程变得更加复杂，因为NN所依赖的磁盘上元数据根据定义是分布的，既分布在对中的两个HA NN上，也分布在QJM用于共享编辑日志存储的JournalNodes上。本文档部分描述在HA设置中使用HDFS升级/完成/回滚功能的过程。

要执行HA升级，管理员必须执行以下操作：

- 正常关闭所有NNs，并安装更新的软件。
- 启动所有JN。请注意，在执行升级、回滚或终结操作时，所有jn都必须运行。如果在运行这些操作时有任何JN关闭，则操作将失败。
- 使用'-upgrade'标志启动其中一个NNs。
- 在开始时，这个NN不会像HA设置中一样进入待机状态。相反，此NN将立即进入活动状态，对其本地存储目录执行升级，并对共享编辑日志执行升级。
- 此时，HA对中的另一个NN将与升级后的NN不同步。为了使其重新同步并再次具有高可用性设置，您应该通过运行带有“-bootstrapStandby”标志的NN来重新引导此NameNode。使用'-upgrade'标志启动第二个NN是错误的。

请注意，如果在终结、回滚或升级之前需要随时重新启动NameNodes，则应正常启动NNs，即不带任何特殊的启动标志。

**查询升级状态**   升级人员在至少一个NNs正在运行时使用“hdfs dfsadmin-upgrade query”命令。对于每个NN，该命令将返回NN升级过程是否完成。

**要完成HA升级**，操作员将在NNs运行且其中一个处于活动状态时使用hdfs dfsadmin-finalize upgrade命令。此时的活动NN将执行共享日志的终结，其本地存储目录包含先前FS状态的NN将删除其本地状态。

**要执行升级的回滚**，应首先关闭两个nn。操作人员应在启动升级过程的NN上运行回滚命令，该命令将在本地dir以及共享日志(NFS或JNS)上执行回滚。启动此NN，并且操作人员应该在另一个NN上运行-bootstrapStandby，以使两个NN与此回滚文件系统状态同步。