# HDFS Federation

## 1. 背景

![](../img/federation-background.gif)

HDFS有两个主要层：

- **NameSpace**
	- 由目录、文件和块组成；
	- 支持所有与命名空间相关的文件系统操作，如创建、删除、修改和列出文件和目录。
- **Block Storage Service**，包括两部分：
	- Block Management文件块管理(运行在Namenode中)
		- 通过DataNode注册与周期性心跳检测，记录DataNode节点信息
		- 处理文件块报告并维护块的位置
		- 支持与块相关的操作，如创建、删除、修改和获取块位置
		- 管理块副本放置、未复制块的块复制，并删除过度复制的块
	- Storage 由Datanodes通过在本地文件系统上存储块并允许读/写访问来提供

之前版本Hadoop的HDFS体系结构只允许整个集群使用一个命名空间，该命名空间由一个NameNode管理。会成为Hadoop集群文件存储的瓶颈，联邦HDFS体系结构通过添加多个NameNode/NameSpace来解决这个限制。

## 2. 多个NameNode/NameSpace

为了水平扩展命名空间服务，联合使用多个独立的NameNode/NameSpace。联邦HDFS中的NameNode整体是联合的，但是其中各个NameNode是相互独立的，管理不同的命名空间，不需要相互协调。Datanodes被所有Namenodes用作块的公共存储空间。每个Datanode都向集群中的所有Namenodes注册。DataNode向NameNode周期性地发送心跳信息和数据块报告，并处理来自Namenodes的命令。

用户可以使用 [ViewFs](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/ViewFs.html) 创建个性化的命名空间视图。ViewFs类似于某些Unix/Linux系统中的客户端挂载表。

![](../img/federation.gif)

### 2.1 Block Pool

块池是属于单个命名空间的一组块，DataNodes存储集群中所有块池的块。每个块池都是独立管理的，这允许各个命名空间为新块生成块ID，而无需与其他命名空间协调。一个NameNode故障不会影响集群中其他NameNode提供服务。

每个NameNode及其Block Pool一起称为命名空间卷(NameSpace Volume)。它是一个独立的管理单位。删除Namenode/NameSpace时，将删除Datanodes中相应的块池。在群集升级期间，每个命名空间卷作为一个单元进行升级。

### 2.2 ClusterID

一个ClusterID标识符用于标识集群中的每个节点。当一个NameNode格式化，将由用户提供或自动生成此标识符。这个ID用于将其他Namenodes格式化到集群中。

### 2.3 主要优点

- 命名空间可伸缩性(Namespace Scalability) 提高命名空间的水平伸缩性，在部署大型集群或集群中有大量小文件时，通过向集群中添加NameNode节点来水平扩展NameSpace。
- 文件系统的吞吐量不再限制与单个NameNode，向集群中添加更多的NameNode来提高文件系统吞吐量
- 隔离(Isolation) 单个Namenode在多用户环境中不提供隔离。例如，实验性应用可能会使Namenode过载，并减慢关键生产型应用的速度。

## 3. 联邦HDFS配置(Federation Configuration)

联邦配置是向后兼容的，允许现有的单个Namenode在配置没有任何更改的情况下继续工作。新配置的设计使得集群中的所有节点都具有相同的配置，而无需根据集群中节点的类型部署不同的配置。

联邦HDFS配置引入了 **NameServiceID**概念，一个NameNode及其SecondaryName/Backup/checkpointer Nodes都属于一个NameService。为了支持单个配置文件，Namenode和secondary/backup/checkpointer配置参数以NameServiceID作为后缀。

### 3.1 配置详情

- 第一步：向集群配置文件中添加 **dfs.nameservices**配置，并使用逗号分隔的nameserviceid列表对其进行配置。这用于集群DataNodes确定NameNodes。

- 第二步：对于每个NameNode及其SecondaryName/Backup/checkpointer，在公共配置文件中添加以下以所属NameServiceID为后缀的配置参数：

	| 守护进程           | 配置参数                                                     |
	| ------------------ | ------------------------------------------------------------ |
	| Namenode           | `dfs.namenode.rpc-address` `dfs.namenode.servicerpc-address` `dfs.namenode.http-address` `dfs.namenode.https-address` `dfs.namenode.keytab.file` `dfs.namenode.name.dir` `dfs.namenode.edits.dir` `dfs.namenode.checkpoint.dir` `dfs.namenode.checkpoint.edits.dir` |
	| Secondary Namenode | `dfs.namenode.secondary.http-address` `dfs.secondary.namenode.keytab.file` |
	| BackupNode         | dfs.namenode.backup.address` `dfs.secondary.namenode.keytab.file |

	以下是两个Namenode的配置示例：

	```xml
	<configuration>
	  <property>
	    <name>dfs.nameservices</name>
	    <value>ns1,ns2</value>
	  </property>
	  <property>
	    <name>dfs.namenode.rpc-address.ns1</name>
	    <value>nn-host1:rpc-port</value>
	  </property>
	  <property>
	    <name>dfs.namenode.http-address.ns1</name>
	    <value>nn-host1:http-port</value>
	  </property>
	   <property>
	    <name>dfs.namenode.secondary.http-address.ns1</name>
	    <value>snn-host1:http-port</value>
	  </property>
	  
	  <property>
	    <name>dfs.namenode.rpc-address.ns2</name>
	    <value>nn-host2:rpc-port</value>
	  </property>
	  <property>
	    <name>dfs.namenode.http-address.ns2</name>
	    <value>nn-host2:http-port</value>
	  </property>
	  <property>
	    <name>dfs.namenode.secondary.http-address.ns2</name>
	    <value>snn-host2:http-port</value>
	  </property>
	</configuration>
	```

### 3.2 格式化Namenodes

**第一步：**使用以下命名格式化Namenode：

```
[hdfs]$ $HADOOP_HOME/bin/hdfs namenode -format [-clusterId <cluster_id>]
```

选择一个不会与环境中的其他群集冲突的唯一的群集id，如果未提供群集id，则会自动生成唯一的群集id。

**第二步：**使用以下命令格式化其他NameNode：

```
[hdfs]$ $HADOOP_HOME/bin/hdfs namenode -format -clusterId <cluster_id>
```

**注意：**步骤2中的群集id必须与步骤1中的群集id相同。如果它们不同，则附加的Namenodes将不属于联合集群的一部分。

###  3.3 从旧版本升级并配置联邦节点

旧版本只支持单个NameNode节点，将群集升级到较新版本，以便在升级期间启用联邦Namenode，您可以按如下方式提供ClusterID：

```
[hdfs]$ $HADOOP_HOME/bin/hdfs --daemon start namenode -upgrade -clusterId <cluster_ID>
```

如果未提供ClusterID，将自动生成集群ID。

### 3.4 向集群中添加一个新Namenode

以下是添加新Namenode步骤：

- 向配置文件中添加`dfs.nameservices`

- 使用NameServiceID为后缀更新配置，0.20版后更改了配置密钥名称。必须使用新的配置参数名称才能使用联合。

- 将新的Namenode相关配置添加到配置文件中

- 将配置文件分发到集群中的其他节点

- 启动新Namenode与Backup/Secondary Nodes

- 通过对群集中的所有Datanode运行以下命令，刷新Datanodes以获取新添加的Namenode：

	```
	[hdfs]$ $HADOOP_HOME/bin/hdfs dfsadmin -refreshNamenodes <datanode_host_name>:<datanode_rpc_port>
	```

## 4. 管理集群

### 4.1 开启/关闭集群

运行以下命令开启集群：

```
[hdfs]$ $HADOOP_HOME/sbin/start-dfs.sh
```

运行以下命令关闭集群：

```
[hdfs]$ $HADOOP_HOME/sbin/stop-dfs.sh
```

这些命令可以在HDFS配置的任何可用节点运行。该命令使用配置来确定集群中的Namenodes，然后在这些节点上启动Namenode进程。Datanodes在workers文件中指定的节点上启动。该脚本可以作为参考以构建自己的脚本以启动和停止集群。

### 4.2 均衡器

均衡器被改为在多个Namenodes情况下工作，可以使用以下命令运行Balancer：

```
[hdfs]$ $HADOOP_HOME/bin/hdfs --daemon start balancer [-policy <policy>]
```

策略参数可以是以下任一项：

- datanode：默认平衡策略，在Datanode级别平衡存储。这类似于以前版本的平衡策略
- blockpool：在blockpool级别平衡存储，同时也在DataNode级别平衡

**注意：**平衡器只会平衡文件数据不会平衡命名空间，有关完整的命令用法，请参阅[balancer](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HDFSCommands.html#balancer)

### 4.3 节点退役

节点退役与之前版本的操作相似，需要退役的节点添加到所有Namenode的exclude文件中。每个Namenode都会停用其块池。当所有Namenodes完成对Datanode的退役时，Datanode被视为退役。

第一步：要将排除文件分发到所有Namenodes，请使用以下命令：

```
[hdfs]$ $HADOOP_HOME/sbin/distribute-exclude.sh <exclude_file>
```

第二步：刷新是有Namenode以获取新的排除节点文件：

```
[hdfs]$ $HADOOP_HOME/sbin/refresh-namenodes.sh
```

上面命令使用HDFS配置来确定集群中已配置的Namenodes，并刷新它们以获取新的排除文件。

### 4.4 集群Web控制台

与Namenode状态网页类似，使用联合HDFS时，可以在http://<any_nn_host:port>/dfsclusterhealth.jsp上监视联合群集。群集中的任何Namenode都可以用来访问此网页。

集群web控制台提供以下信息：

- 群集摘要：显示整个群集的文件数、块数、配置的总存储容量以及可用和已用存储
- 一个Namenodes列表及列表中Namenode摘要，包括每个Namenode的文件数、块数、丢失块数以及活动和非活动Datanode数。还提供了访问每个Namenode的web UI的链接。
- 数据节点的退役状态