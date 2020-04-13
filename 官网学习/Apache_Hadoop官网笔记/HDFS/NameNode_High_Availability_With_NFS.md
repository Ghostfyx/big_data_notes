# HDFS High Availability

## 1.  主旨与概述

本指南概述了HDFS高可用(HA)特性，以及如何将NFS用于NameNodes共享存储，去配置和管理HA HDFS集群。

本文档假设读者对HDFS集群中的一般组件和节点类型有一个大致的了解。有关详细信息，请参阅HDFS体系结构指南。

------

**使用Quorum Journal Manager还是常规共享存储**

本指南讨论如何使用共享NFS目录配置和使用HDFS HA，以便在活动和备用NameNodes之间共享编辑日志。有关如何使用Quorum Journal Manage而不是NFS配置HDFS HA的信息，请参阅High_Availablity_Using_QJM。

## 2. 背景介绍

在Hadoop 2.0.0之前，NameNode是HDFS集群中的单点故障(SPOF)。每个群集都有一个NameNode，如果该机器或进程不可用，则整个群集将不可用，直到NameNode重新启动或在另一台机器上重新部署。

单NameNode以下两个主要方面影响了HDFS群集总体的可用性：

- 在意外事件(例如如机器崩溃)的情况下，集群将不可用，直到管理员重新启动NameNode。
- 机器日常维护(如NameNode计算机上的软件或硬件升级)，将导致群集停机时间窗口。

HDFS的高可用性特性是通过同一集群中运行两个(从3.0.0开始超过两个)冗余NameNodes的选项来解决上述问题，NameNode采用主/从配置，并具有热备用。这允许在NameNode节点机器崩溃的情况下快速故障转移到新的NameNode，或者为了日常维护而由管理员启动的正常故障转移。

## 3. Hadoop HA集群结构

在典型的HA集群中，有两台或多台独立的NameNodes节点。在任何时间点，都有一个NameNode处于活动状态，其他的处于备用状态。活动的NameNode负责集群中的所有客户端操作，而备用节点只是充当worker，保持集群状态的更新，以便在必要时提供快速故障转移。

为了使备用节点的状态与活动节点保持同步，当前的实现要求这些节点能够访问共享存储设备上的目录（例如，从NAS装载的NFS）。在未来的版本中，这一限制可能会放宽。

当活动节点执行任何命名空间修改时，活跃的NameNode会将修改记录持久化记录到存储在共享目录中的编辑日志文件中。备用节点不断扫描这个目录，当NFS中的编辑日志发生修改，备用节点将修改内容应用于本地命名空间。在发生故障转移时，在备用NameNode状态变更前，确保已从共享存储中读取所有编辑日志。这样可以在发生故障转移之前完全同步命名空间状态。

为了提供快速故障转移，还需要备用节点具有有关集群中文件块位置的最新信息。为了实现这一点，DataNode被配置有所有NameNodes(Active与Standby节点)的位置，并向所有NameNodes发送块位置信息和心跳信号。

一次只有一个NameNode处于活动状态对于HA集群的正确操作至关重要。否则，名称空间状态将很快在两者之间出现差异，从而可能导致数据丢失或其他不正确的结果(这种情况称为**脑裂**)。为了一个Active NameNode并防止分割情况发生，管理员必须为共享存储配置至少一个隔离方法。在故障转移期间，如果无法保证上一个活动NameNode状态是否已经改变，则**防护进程**负责切断上一个活动节点对共享编辑存储的访问。这将防止它对命名空间进行任何进的编辑，从而允许新的活动项安全地继续进行故障转移。

## 4. 硬件资源

为了部署HA集群，应该准备以下资源：

- **NameNode machines**：运行活动和备用NameNode节点的机器应该具有相同的硬件设施，每台NameNode节点机器配置与非HA集群中使用的硬件一致。

- **Shared storage**：需要准备允许NameNode机器节点读/写的共享的存储目录。共享存储目录通常是一个支持NFS的远程文件管理器，安装在每个NameNode机器上。目前只支持一个共享编辑目录。因此，系统的可用性受此共享编辑目录的可用性限制。为了消除所有单点故障，需要对共享编辑目录进行冗余。具体地说，到存储器的多个网络路径以及存储本身的冗余(磁盘、网络和电源)。因此，建议共享存储服务器是高质量的专用NAS设备，而不是简单的Linux服务器。

**注意：**在Hadoop HA集群中，备用NameNodes还执行命名空间状态的检查点，因此不必在HA集群中运行SecondaryNameNode、CheckpointNode或BackupNode。实际上如果你运行这些服务的话会出错。

## 5. 部署

### 5.1 部署概述

与Federation configuration类似，HA configuration会向后兼容(**backward compatible**) 并且已经存在的单个NameNode继续工作，不用更改任何配置。新配置的设计使得集群中的所有节点可能具有相同的配置，而无需根据节点的类型将不同的配置文件部署到不同的节点。

与HDFS Federation相同，HA集群也用**nameservice ID**来标识实际上可能由多个NameNodes组成的单个HDFS实例。此外，NameNode ID的概念被引入到HA集群中。群集中每个不同的NameNode都有不同的NameNode ID作为区分。为了支持所有NameNode使用单个配置文件，每一个 Namenode和Secondary Namenode/BackupNode/Checkpointer 加上以自己对应的NameServiceID参数为后缀。

### 5.2 配置详情

为部署HA集群，需要将几个配置加入集群的`hdfs-site.xml`配置文件。

这些配置在配置文件中的顺序不会影响集群运行，但是 **dfs.nameservices** and **dfs.ha.namenodes.[nameservice ID]** 的值是决定后面这些配置的关键，应该在其余配置选项之前这些值。

- **dfs.nameservices**：新NameService的名称。为nameService选择一个逻辑名称，例如“mycluster”，它将同时用于配置和集群中HDFS绝对路径的权限组件。

	**注意：**如果还使用HDFS Federation配置，则此配置还应包括其他nameservice列表，之间使用逗号分隔。

	```xml
	<property>
	  <name>dfs.nameservices</name>
	  <value>mycluster</value>
	</property>
	```

- **dfs.ha.namenodes.[nameservice ID]** ：nameservices中每个namenode的唯一标识符。使用逗号分隔的NameNode id列表进行配置，用于DataNodes确定集群中的所有NameNode。例如，如果使用mycluster作为nameserviceID，并且使用nn1、nn2和nn3作为NameNode的ID，则可以这样配置：

	```xml
	<property>
	  <name>dfs.ha.namenodes.mycluster</name>
	  <value>nn1,nn2,nn3</value>
	</property>
	```

	**注意：**Hadoop HA集群的nameNode的最少数量为2，但是可以配置更多，由于通信开销，建议不超过5个(推荐3个NameNode)。

- **dfs.namenode.rpc-address.[nameservice ID].[name node ID]**：要监听的每个NameNode的全限定RPC地址。对于之前配置的NameNodeID，设置NameNode进程的完整地址和IPC端口。需要对每个NameNode的RPC地址单独配置：

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

	**注意：**也可以类似配置**NameNodeService**的RPC地址——**servicerpc-address**

- **dfs.namenode.http-address.[nameservice ID].[name node ID]**  要监听的每个NameNode的全限定HTTP地址。与配置NameNode的rpc-address类似，需要为每个NameNode配置http-rddress

	```xml
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

	**注意：**如果启用了Hadoop的安全功能，那么还应该为每个NameNode设置https地址。

- **fs.namenode.shared.edits.dir**：共享存储目录的位置。可以配置远程共享编辑目录的路径，备用NameNodes使用该目录来更新活动NameNode所做的所有文件系统更改。只需要配置一个共享存储目录，此目录在NameNode允许读/写。这个值被设置为NameNode的绝对路径：

	``` xml
	<property>
	  <name>dfs.namenode.shared.edits.dir</name>
	  <value>file:///mnt/filer1/dfs/ha-name-dir-shared</value>
	</property>
	```

- **dfs.client.failover.proxy.provider.[nameservice ID]** HDFS客户端联系活动NameNode的Java类。配置DFS客户端用于确定活动NameNode和为客户端服务NameNode的ava类的名称，当前Hadoop提供的两个实现ConfiguredFailoverProxyProvider和RequestHedgingProxyProvider。ConfiguredFailoverProxyProvider同时调用所有namenodes来确定活动的namenode，在随后的请求中，调用活动的namenode，直到发生故障转移。因此，除非使用自定义代理提供程序，否则请使用其中一个，例如：

	```xml
	<property>
	  <name>dfs.client.failover.proxy.provider.mycluster</name><value>org.apache.hadoop.hdfs.server.namenode.ha.ConfiguredFailoverProxyProvider</value>
	</property>
	```

- **dfs.ha.fencing.methods**  在故障转移期间用于隔离活动NameNode的脚本或Java类的列表。为了保证系统的正确性，在给定的时间内只有一个NameNode处于活动状态。因此，在故障转移期间，首先确保活动NameNode处于待机状态，或者进程已终止，然后再将另一个NameNode转换为活动状态。为此，必须至少配置一个fenching methods。它们被配置为一个回车分隔列表，将按顺序尝试该列表，直到其中一个指示围栏已成功。Hadoop附带两种方法：shell和sshfence。有关实现自定义围栏方法的信息，请参见**org.apache.hadoop.ha.NodeFencer**类。

	- **sshfence** SSH到活动NameNode并终止进程。sshfence选项使用SSH登陆到活动NameNode，并使用fuser终止监听服务TCP端口的进程。需要必须能够在不提供密码短语的情况下通过SSH连接到目标节点。因此，还必须配置dfs.ha.fencing.ssh.private-key-files选项，该选项是ssh私钥文件的逗号分隔列表。例如：

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

		或者，可以配置非标准用户名或端口来执行SSH。还可以为SSH配置一个超时(以毫秒为单位)，在该超时之后，此隔离方法将被视为失败。其配置如下：

		```xml
		<property>
		  <name>dfs.ha.fencing.methods</name>
		  <value>sshfence([[username][:port]])</value>
		</property>
		<property>
		  <name>dfs.ha.fencing.ssh.connect-timeout</name>
		  <value>30000</value>
		</property>
		```

	- **shell** 运行任意shell命令来隔离活动的NameNode。以shell方式运行fencing method方法如下：

		```xml
		<property>
		  <name>dfs.ha.fencing.methods</name>
		  <value>shell(/path/to/my/script.sh arg1 arg2 ...)</value>
		</property>
		```

		 ‘(’ 和‘)’之间的字符串直接被传递给bash shell，字符串不能包含任何右括号。shell命名将在包含所有当前Hadoop配置变量的环境中运行，配置键中的“.”字符被替换为"_"。所使用的配置已将任何特定于namenode的配置变为其通用格式"-"，例如：**dfs_namenode_rpc-address** 将会包含NameNode的RPC地址。

		此外，还可以用以下变量设置要隔离的目标NameNode：

		| 变量名称              | 变量描述                                  |
		| --------------------- | ----------------------------------------- |
		| $target_host          | hostname of the node to be fenced         |
		| $target_port          | IPC port of the node to be fenced         |
		| $target_address       | the above two, combined as host:port      |
		| $target_nameserviceid | the nameservice ID of the NN to be fenced |
		| $target_namenodeid    | the namenode ID of the NN to be fenced    |

		这些环境变量也可以用作shell命令本身的替换。例如：

		```xml
		<property>
		  <name>dfs.ha.fencing.methods</name>
		  <value>shell(/path/to/my/script.sh --nameservice=$target_nameserviceid $target_host:$target_port)</value>
		</property>
		```

		如果shell命令返回退出代码0，则fencing设置成功。

		**注意：**fencing method没有实现超时设置，如果需要超时时间，应该在Shell命名脚本本身实现(例如，使用子脚本在几秒钟内杀死其父脚本)。

- **fs.defaultFS** 当Hadoop文件系统客户端在未给定任何路径前缀时，使用的默认路径前缀。HA集群中，为了使用HA文件系统，可以修改Hadoop客户机默认文件路径前缀，如果之前使用“mycluster”作为nameservice ID，这将是所有HDFS路径的authority部分的值。在core-site.xml文件中可以这样配置：

	```xml
	<property>
	  <name>fs.defaultFS</name>
	  <value>hdfs://mycluster</value>
	</property>
	```

### 5.3 部署详情

在所有配置信息都设置完毕后，必须首先同步两个HA NameNodes的磁盘元数据：

- 如果是要设置一个新的HDFS集群，应该首先在其中一个NameNodes上运行format命令：HDFS namenode-format。
- 如果已经格式化NameNode或要将为启用HA Hadoop集群启用高可用，现在需要将运行中NameNode的元数据目录复制到另一个(新加入)的NameNode机器上。通过在未格式化的namenode上运行命令**hdfs namenode-bootstrapStandby**来取消格式化的NameNode。运行此命令还将确保共享编辑目录(由dfs.namenode.shared.edits.dir配置)包含足够的编辑事务，以便能够启动两个NameNodes。
- 如果正在将非HA NameNode转换为HA，则应运行命令`hdfs NameNode-initializeSharedEdits`，该命令将使用本地NameNode编辑目录中的编辑数据初始化共享存储目录。

此时，可以启动所有HA NameNodes，就像通常启动NameNode一样。通过每个NameNode的配置HTTP地址，可以分别访问它们的web页面。应该注意到，在配置的地址旁边将是NameNode的HA状态(standby或active)。当一个HA NameNode启动时，它最初都处于standby。

### 5.4 管理员命令

现在高可以用的NameNode已经配置并启动，可以执行一些命令来管理HA HDFS集群。具体来说，应该熟悉`hdfs haadmin`命令的所有子命令。在没有任何附加参数的情况下运行此命令将显示以下用法信息：

```
Usage: DFSHAAdmin [-ns <nameserviceId>]
    [-transitionToActive <serviceId>]
    [-transitionToStandby <serviceId>]
    [-failover [--forcefence] [--forceactive] <serviceId> <serviceId>]
    [-getServiceState <serviceId>]
    [-getAllServiceState]
    [-checkHealth <serviceId>]
    [-help <command>]
```

本指南描述了这些子命令的高级用法。对于每个子命令的特定用法信息，可以运行**hdfs haadmin-help< command>**：

- **transitionToActive** 与**transitionToStandby**：将制定NameNode转换为活跃/备用状态。

	这些子命令分别使给定的NameNode转换为Active或Standby状态。这些命令不试图执行任何阻拦，因此很少使用。相反，更推荐使用hdfs haadmin-failover子命令。

- **failover** ：在两个NameNode之间启动故障转移。

	此子命令将开启从第一个NameNode到第二个NameNode的故障转移。如果第一个NameNode处于待机状态，则此命令只需将第二个NameNode转换为活动状态而不会出错。如果第一个NameNode处于活动状态，则会尝试将正常转换为待机，如果转换失败，将按顺序尝试隔离方法(由**dfs.ha.fencing.methods**配置)，直到成功为止。只有在此进程之后，第二个NameNode才会转换为活动状态。如果没有fencing method成功，则第二个NameNode将不会转换为活动状态，并返回错误。

- **getServiceState**：确定制定NameNode状态。连接到提供的NameNode以确定其当前状态，并将standby或active打印到标准输出。此子命令可用于cron作业或监视脚本，作业或监视脚本需要根据NameNode当前状态而采取不同的行为。

- **getAllServiceState** ：返回所有NameNode当前状态。

	连接到已配置的NameNodes以确定当前状态，将standby或active打印到STDOUT。

- **checkHealth**：检查给定NameNode的健康状况。

	检查给定namenode健康情况。连接到指定NameNode以检查其运行状况，NameNode能够对自身执行一些诊断，包括检查内部服务是否按预期运行。如果NameNode正常，此命令将返回0，否则返回非0。可以使用此命令进行监视。

## 6. 自动故障转移

### 6.1 简介

以上部分描述如何配置手动故障转移。在该模式下，即使活动节点发生故障，系统也不会自动触发从活动节点到secondaryNamenode的故障转移。本节介绍如何配置和部署自动故障转移。

### 6.2 组件

自动故障转移为HDFS部署添加了两个新组件：ZooKeeper quorum和ZKFailoverController进程(缩写为ZKFC)。

Apache ZooKeeper是一个高度可用的服务，用于维护少量的协调数据、通知客户端数据的更改以及监视机器故障。自动HDFS故障转移的实现依赖于ZooKeeper完成以下任务：

- **故障检测**  群集中的每个NameNode节点都在ZooKeeper中维护一个持久session。如果节点机器崩溃，ZooKeeper会话将过期，从而通知其他NameNode应触发故障转移。
- **活跃NameNode选举**  ZooKeeper提供了一个简单的机制来独占地选择一个节点为Active。如果当前活动的NameNode崩溃，另一个节点可能会在ZooKeeper中使用一个特殊的独占锁，指示它应该成为下一个活动节点。

ZKFailoverController(ZKFC)是一个Hadoop新组件，它是一个ZooKeeper客户端，用于监视和管理NameNode的状态。运行NameNode的每台机器也运行一个ZKFC，ZKFC负责：

- **运行状况监视**  ZKFC使用运行状况检查命令定期ping其本地NameNode。只要NameNode及时响应，ZKFC就认为该节点健康。如果节点已崩溃、冻结或其他方式进入不正常状态，ZKFailoverController会将其标记为不正常。
- **ZooKeeper会话管理**  当本地NameNode正常时，ZKFC在ZooKeeper中保持会话打开。如果本地NameNode处于活动状态，它同时也持有一个“排他锁”znode；如果会话过期，锁节点将被自动删除。
- **ZooKeeper选举机制**  如果本地NameNode是健康的，并且ZKFC看到当前没有其他节点持有锁znode，那么它自己将尝试获取锁。如果成功，则它“赢得了选举”，并负责运行故障转移以使其本地NameNode处于活动状态。故障转移过程类似于上面描述的手动故障转移：首先，如果需要，前一个active节点被隔离，然后本地NameNode转换为active状态。

### 6.3 部署Zookeeper

在典型的部署中，ZooKeeper守护进程配置为在三个或五个节点上运行。由于ZooKeeper本身具有轻资源需求，因此可以将ZooKeeper节点与HDFS NameNode和SecondaryNameNode配置在同一硬件上。许多管理员选择将第三个ZooKeeper进程部署在与YARN ResourceManager相同的节点上。但是，建议将ZooKeeper节点配置为将其数据与HDFS元数据存储在不同的磁盘驱动器上，以获得最佳性能和隔离。

### 6.4 部署前

在开始配置自动故障转移之前，应关闭群集。因为，群集运行时，无法从手动故障转移设置过渡到自动故障转移设置。

### 6.5 配置失败自动转移

自动故障转移的配置需要在配置文件中添加两个配置。

1. 在hdfs-site.xml文件中添加以下配置，为集群开启故障自动转移：

	```xml
	<property>
	   <name>dfs.ha.automatic-failover.enabled</name>
	   <value>true</value>
	 </property>
	```

2. 在core-site.xml中添加一下配置，用于执行运行Zookeeper服务的地址和端口。

	```xml
	<property>
	   <name>ha.zookeeper.quorum</name>
	   <value>zk1.example.com:2181,zk2.example.com:2181,zk3.example.com:2181</value>
	 </property>
	```

与文档前面描述的参数一样，这些设置可以在每个nameservice的基础上配置，方法是用nameserviceID作为配置键的后缀。例如，在启用联合的集群中，通过设置dfs.ha.automatic-failover.enabled.my-nameservice-id，只能显式地为其中一个nameservices启用自动故障转移。

还可以设置其他几个配置参数来控制自动故障转移的行为；但是，大多数安装都不需要这些参数。有关详细信息，请参阅配置密钥特定文档。

### 6.6 ZooKeeper初始化HA状态

添加配置后，下一步是在ZooKeeper中初始化所需状态。可以通过从一个NameNode主机运行以下命令来执行此操作。

```
[hdfs]$ $HADOOP_HOME/bin/zkfc -formatZK
```

这将在ZooKeeper中创建一个znode，自动故障转移系统将在其中存储其数据。

### 6.7 使用start-dfs.sh脚本启动集群

由于在配置中启用了自动故障转移，start-dfs.sh脚本现在将在运行NameNode的机器上上自动启动ZKFC守护进程。当各个NameNode节点ZKFC启动时，它们将自动选择一个NameNode为活跃状态。

### 6.8 手动启动集群

如果手动管理集群上的服务，则需要在运行NameNode的每台机器上手动启动zkfc守护进程。可以通过运行以下命令启动守护程序：

```bash
[hdfs]$ $HADOOP_HOME/bin/hdfs --daemon start zkfc
```

### 6.9 安全访问ZooKeeper

如果运行的是安全群集，则可能需要确保ZooKeeper中存储的信息也是安全的。这可防止恶意客户端修改ZooKeeper中的元数据或可能触发错误故障转移。

为了保证Zookeeper中的数据安全，需要在core-site.xml配置以下选项：

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

```
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

### 6.10 验证自动故障转移

设置自动故障转移后，应测试其操作。为此，首先找到活动的NameNode。通过访问NameNode web界面来判断哪个节点处于活动状态，每个节点在页面顶部报告其HA状态。

找到活动的NameNode后，可能会导致该节点出现故障。例如，可以使用kill-9< pid of NN>来模拟JVM崩溃。或者，您可以重启机器或拔下其网络接口以模拟不同类型的中断。触发要测试的中断后，另一个NameNode应在几秒钟内自动变为活动状态。检测故障并触发故障转移所需的时间取决于ha.zookeeper.session-timeout.ms的配置，但默认为5秒。

如果测试不成功，则可能是配置错误。检查zkfc守护进程和NameNode守护进程的日志，以便进一步诊断问题。

## 7. 自动故障转移 FAQ

