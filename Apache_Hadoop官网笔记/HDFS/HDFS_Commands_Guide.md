# HDFS Command Guide

# 1. OverView

所有HDFS命令都由bin/ HDFS脚本调用。不带任何参数运行hdfs脚本将打印命令的描述。

用法：`hdfs [SHELL_OPTIONS] COMMAND [GENERIC_OPTIONS] [COMMAND_OPTIONS]`

Hadoop有一个选项解析框架，用于解析通用选项和运行类。

| COMMAND_OPTIONS           | Description                                                  |
| ------------------------- | ------------------------------------------------------------ |
| SHELL_OPTIONS             | 在[Commands Manual](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/CommandsManual.html#Shell_Options)页面可查看所有SHELL命令集合 |
| GENERIC_OPTIONS           | 多个命令支持的公共选项集，详情可查看[Commands Manual](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/CommandsManual.html#Generic_Options) |
| COMMAND & COMMAND_OPTIONS | 用户命令和管理命令                                           |

# 2. User Commands

用户对Hadoop集群操作的命令。

### 2.1 ClassPath

用法：`hdfs classpath [--glob |--jar <path> |-h |--help]`

| COMMAND_OPTION | Description               |
| :------------- | :------------------------ |
| `--glob`       | expand wildcards          |
| `--jar` *path* | hadoop jar或者lib的类路径 |
| `-h`, `--help` | 打印帮助                  |

该命令主要为打印Hadoop jar中所加载的类路径。如果不带参数调用，则打印由命令脚本中所有配置的类路径。路径中包含分隔符。另外一个选项是，将classpath写入到mainfest中，生成一个jar包，这个命令是很有用的，尤其是当类太多时，控制台无法完整打印时。

### 2.2 dfs

用法：`hdfs dfs [COMMAND [COMMAND_OPTIONS]]`

在Hadoop支持的文件系统上运行文件系统命令，更多信息可以查看[File System Shell Guide](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/FileSystemShell.html).

### 2.3 envvars

用法： `hdfs envvars`

显示Hadoop的运算环境，是Hadoop 3x版本的新命令。

### 2.4 fetchdt

 用法： `hdfs fetchdt  `

| COMMAND_OPTION        | Description                                            |
| :-------------------- | :----------------------------------------------------- |
| `--webservice` NN_Url | 与NN联系的Url（以http或https开头）                     |
| `--renewer name`      | 委托令牌的名称                                         |
| `--cancel`            | 取消委托令牌                                           |
| `--renew`             | 更新委托令牌。必须使用- renew *name*选项获取委托令牌。 |
| `--print`             | Print the delegation token                             |
| `token_file_path`     | 委托令牌存储位置                                       |

### 2.5 fask

用法：

```
hdfs fsck <path>
          [-list-corruptfileblocks |
          [-move | -delete | -openforwrite]
          [-files [-blocks [-locations | -racks | -replicaDetails | -upgradedomains]]]
          [-includeSnapshots] [-showprogress]
          [-storagepolicies] [-maintenance]
          [-blockId <blk_Id>]
```

| COMMAND_OPTION                       | Description                                                  |
| :----------------------------------- | :----------------------------------------------------------- |
| *path*                               | 文件块检查开始路径                                           |
| `-delete`                            | 删除检查完毕的块                                             |
| `-files`                             | 打印已经检查的文件信息                                       |
| `-files` `-blocks`                   | 打印块报告                                                   |
| `-files` `-blocks` `-locations`      | 打印数据块存储位置                                           |
| `-files` `-blocks` `-racks`          | 打印数据节点位置的网络拓扑                                   |
| `-files` `-blocks` `-replicaDetails` | 打印每个数据块复本的详情                                     |
| `-files` `-blocks` `-upgradedomains` | Print out upgrade domains for every block.                   |
| `-includeSnapshots`                  | 如果给定路径指示snapshottable目录或其下有snapshottable目录，则包括快照数据 |
| `-list-corruptfileblocks`            | 打印出丢失的块及其所属文件的列表                             |
| `-move`                              | 将损坏的文件移到/丢失+找到                                   |
| `-openforwrite`                      | 打印正要写入的文件                                           |
| `-showprogress`                      | 打印输出进度。默认关闭。                                     |
| `-storagepolicies`                   | 打印块的存储策略摘要                                         |
| `-maintenance`                       | 打印维护状态节点详细信息                                     |
| `-blockId`                           | Print out information about the block.                       |

### 2.6 getconf

用法：

```
hdfs getconf -namenodes
hdfs getconf -secondaryNameNodes
hdfs getconf -backupNodes
hdfs getconf -journalNodes
hdfs getconf -includeFile
hdfs getconf -excludeFile
hdfs getconf -nnRpcAddresses
hdfs getconf -confKey [key]
```

| COMMAND_OPTION        | Description                                              |
| :-------------------- | :------------------------------------------------------- |
| `-namenodes`          | 获取集群中namenode列表                                   |
| `-secondaryNameNodes` | 获取集群中secondaryNameNode列表                          |
| `-backupNodes`        | 获取集群中备份节点列表                                   |
| `-journalNodes`       | 获取群集中日志节点的列表                                 |
| `-includeFile`        | 获取定义数据节点白名单(允许接入集群的数据节点)的文件路径 |
| `-excludeFile`        | 获取定义数据节点黑名单(停用的数据节点)的文件路径         |
| `-nnRpcAddresses`     | 获取namenode rpc地址                                     |
| `-confKey` [key]      | gets a specific key from the configuration               |

### 2.7 groups

用法： `hdfs groups [username ...]`

返回给定一个或多个用户名的组信息

### 2.8 httpfs

用法： `hdfs httpfs`

运行HttpFS服务器，即HDFS HTTP网关。

### 2.9 lsSnapshottableDir

用法： `hdfs lsSnapshottableDir [-help]`

获取快照表目录列表。当它作为超级用户运行时，它返回所有快照表目录。否则，它将返回当前用户拥有的目录。

### 2.10 jmxget

用法：``hdfs jmxget [-localVM ConnectorURL | -port port | -server mbeanserver | -service service]`

| COMMAND_OPTION            | Description                                                  |
| :------------------------ | :----------------------------------------------------------- |
| `-help`                   | print help                                                   |
| `-localVM ConnectorURL`   | 连接到同一台计算机上的虚拟机                                 |
| `-port mbean server port` | 指定mbean服务器端口，如果缺少，它将尝试连接到同一个虚拟机中的mbean服务器 |
| `-server`                 | 指定mbean服务器(默认情况下为本地主机)                        |
| `-service NameNode`       | 指定jmx服务。默认情况下为NameNode。                          |

从一个服务中转储JMX信息。

### 2.11 oev(offline edits viewer)

用法：`hdfs oev [OPTIONS] -i INPUT_FILE -o OUTPUT_FILE`

#### Required command line arguments:

| COMMAND_OPTION          | Description                                                  |
| :---------------------- | :----------------------------------------------------------- |
| `-i`,`--inputFile arg`  | 要处理editsLog文件，xml(不区分大小写)扩展名表示xml格式，任何其他文件名表示二进制格式 |
| `-o`,`--outputFile arg` | 输出文件的名称。如果指定的文件存在，它将被覆盖，文件的格式由-p选项决定 |

#### Optional command line arguments:

| COMMAND_OPTION         | Description                                                  |
| :--------------------- | :----------------------------------------------------------- |
| `-f`,`--fix-txids`     | 重新对输入的事务ID编号，用于消除间隙或无效的事务ID           |
| `-h`,`--help`          | Display usage information and exit                           |
| `-r`,`--recover`       | 读取二进制编辑日志时，请使用恢复模式。这将会跳过编辑日志中损坏的部分 |
| `-p`,`--processor arg` | 选择要应用于图像文件的处理器类型， 当前支持的处理器类型: binary (Hadoop使用的本机二进制格式), xml (默认，XML格式), stats (打印编辑日志的统计信息) |
| `-v`,`--verbose`       | 更详细的输出，打印输入和输出文件名，对于写入文件的处理器，也输出到屏幕。对于大型fsimage文件，这将显著增加处理时间(默认值为false) |

Hadoop离线编辑日志查看器.  [Offline Edits Viewer Guide](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsEditsViewer.html)查看相信信息。

### 2.12 oiv(Offline Image Viewe)

用法：`hdfs oiv [OPTIONS] -i INPUT_FILE`

#### Required command line arguments:

| COMMAND_OPTION                | Description                                                  |
| :---------------------------- | :----------------------------------------------------------- |
| `-i`|`--inputFile input file` | 指定要处理的fsimage文件（或XML文件，如果已经使用ReverseXML处理器） |

#### Optional command line arguments:

| COMMAND_OPTION                  | Description                                                  |
| :------------------------------ | :----------------------------------------------------------- |
| `-o`,`--outputFile output file` | 如果指定的输出处理器生成文件名，请指定输出文件名。如果指定的文件已经存在，它将被自动覆盖(默认情况下输出到stdout)。如果输入文件是一个XML文件，它还会创建一个< output file>.md5。 |
| `-p`,`--processor processor`    | 指定处理fsimage文件的处理器，当前处理器有：Web(默认)、XML、Delimited、FileDistribution和ReverseXML。 |
| `-addr address`                 | 指定要侦听的地址(主机：端口(默认本地主机：5978)。此选项用于Web处理器 |
| `-maxSize size`                 | 指定要处理的文件字节数(默认128GB)，可选范围是[0, maxSize]，此选项用于文件分发处理器 |
| `-step size`                    | 以字节为单位指定分发的粒度(默认为2 MB)。此选项用于文件分发处理器。 |
| `-format`                       | 以人类可读的方式格式化输出结果，而不是设置字节数(默认false)。此选项用于FileDistribution Processor |
| `-delimiter arg`                | 用于Delimited processor的分隔字符串                          |
| `-t`,`--temp temporary dir`     | 使用temporary dir缓存中间结果以生成分隔的输出，如果没有设置，Delimited processor 在输出文本之前于内存中构造命名空间 |
| `-h`,`--help`                   | Display the tool usage and help information and exit.        |

### 2.13 snapshotDiff

用法：`hdfs snapshotDiff <path> <fromSnapshot> <toSnapshot>`

比较HDFS快照之间的差异。

### 2.14 version

用法：`hdfs version`

打印hdfs版本号。

## 3. Administration Commands

Hadoop集群管理员使用的命令。

### 3.1 balancer

用法：

```
 hdfs balancer
          [-policy <policy>]
          [-threshold <threshold>]
          [-exclude [-f <hosts-file> | <comma-separated list of hosts>]]
          [-include [-f <hosts-file> | <comma-separated list of hosts>]]
          [-source [-f <hosts-file> | <comma-separated list of hosts>]]
          [-blockpools <comma-separated list of blockpool ids>]
          [-idleiterations <idleiterations>]
          [-runDuringUpgrade]
```

| COMMAND_OPTION                                               | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `-policy` < policy>                                          | `datanode`(默认): 如果集群上每个datanode都数据平衡，则集群平衡blockpool: 如果每个数据节点中的每个块池都是平衡，则集群平衡 |
| `-threshold` <threshold>                                     | 磁盘容量的百分比。这将覆盖默认阈值                           |
| `-exclude -f` < hosts-file> \| < comma-separated list of hosts> | 排除平衡器处理的指定数据节点                                 |
| `-include -f` < hosts-file> \| < comma-separated list of hosts> | 指定平衡器要处理的数据节点                                   |
| `-source -f` < hosts-file> \| < comma-separated list of hosts> | 仅选取指定的数据节点作为源节点。                             |
| `-blockpools` < comma-separated list of blockpool ids>       | 平衡器将只在包含在此列表中的块池上运行                       |
| `-idleiterations` < iterations>                              | Maximum number of idle iterations before exit. This overwrites the default idleiterations(5). |
| `-runDuringUpgrade`                                          | 是否在正在进行的HDFS升级期间运行平衡器。这通常是不需要的，因为它不会影响过度利用机器上的使用空间。 |
| `-h`|`--help`                                                | Display the tool usage and help information and exit.        |

运行群集平衡实用程序。管理员只需按Ctrl-C即可停止重新平衡过程。有关详细信息，请参见[Balancer](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsUserGuide.html#Balancer) 。

注意：blockpool平衡策略比datanode更严格。

除了上述命令选项外，还从2.7.0开始引入了固定功能，以防止平衡器/移动器移动某些副本。此固定功能在默认情况下已禁用，并且可以通过配置属性“dfs.datanode.block pinning.enabled”启用。启用时，此功能仅影响写入create()中的首选节点的块。对于HBase regionserver等应用程序，在想要维护数据局部性时，此功能非常有用。

### 3.2 cacheadmin

用法

```
hdfs cacheadmin [-addDirective -path <path> -pool <pool-name> [-force] [-replication <replication>] [-ttl <time-to-live>]]
hdfs cacheadmin [-modifyDirective -id <id> [-path <path>] [-force] [-replication <replication>] [-pool <pool-name>] [-ttl <time-to-live>]]
hdfs cacheadmin [-listDirectives [-stats] [-path <path>] [-pool <pool>] [-id <id>]]
hdfs cacheadmin [-removeDirective <id>]
hdfs cacheadmin [-removeDirectives -path <path>]
hdfs cacheadmin [-addPool <name> [-owner <owner>] [-group <group>] [-mode <mode>] [-limit <limit>] [-maxTtl <maxTtl>]]
hdfs cacheadmin [-modifyPool <name> [-owner <owner>] [-group <group>] [-mode <mode>] [-limit <limit>] [-maxTtl <maxTtl>]]
hdfs cacheadmin [-removePool <name>]
hdfs cacheadmin [-listPools [-stats] [<name>]]
hdfs cacheadmin [-help <command-name>]
```

### 3.3 datanode

用法：`hdfs datanode [-regular | -rollback | -rollingupgrade rollback]`

| COMMAND_OPTION             | Description                                                  |
| :------------------------- | :----------------------------------------------------------- |
| `-regular`                 | 常规方式启动datanode(默认选项).                              |
| `-rollback`                | 将datanode回滚到以前的版本。这应该在停止datanode并分发旧的hadoop版本之后使用。 |
| `-rollingupgrade` rollback | 回滚升级操作                                                 |

启动HDFS数据节点。

### 3.4 dfsadmin

用法：

```
hdfs dfsadmin [-report [-live] [-dead] [-decommissioning] [-enteringmaintenance] [-inmaintenance]]
hdfs dfsadmin [-safemode enter | leave | get | wait | forceExit]
hdfs dfsadmin [-saveNamespace [-beforeShutdown]]
hdfs dfsadmin [-rollEdits]
hdfs dfsadmin [-restoreFailedStorage true |false |check]
hdfs dfsadmin [-refreshNodes]
hdfs dfsadmin [-setQuota <quota> <dirname>...<dirname>]
hdfs dfsadmin [-clrQuota <dirname>...<dirname>]
hdfs dfsadmin [-setSpaceQuota <quota> [-storageType <storagetype>] <dirname>...<dirname>]
hdfs dfsadmin [-clrSpaceQuota [-storageType <storagetype>] <dirname>...<dirname>]
hdfs dfsadmin [-finalizeUpgrade]
hdfs dfsadmin [-rollingUpgrade [<query> |<prepare> |<finalize>]]
hdfs dfsadmin [-upgrade [query | finalize]
hdfs dfsadmin [-refreshServiceAcl]
hdfs dfsadmin [-refreshUserToGroupsMappings]
hdfs dfsadmin [-refreshSuperUserGroupsConfiguration]
hdfs dfsadmin [-refreshCallQueue]
hdfs dfsadmin [-refresh <host:ipc_port> <key> [arg1..argn]]
hdfs dfsadmin [-reconfig <namenode|datanode> <host:ipc_port> <start |status |properties>]
hdfs dfsadmin [-printTopology]
hdfs dfsadmin [-refreshNamenodes datanodehost:port]
hdfs dfsadmin [-getVolumeReport datanodehost:port]
hdfs dfsadmin [-deleteBlockPool datanode-host:port blockpoolId [force]]
hdfs dfsadmin [-setBalancerBandwidth <bandwidth in bytes per second>]
hdfs dfsadmin [-getBalancerBandwidth <datanode_host:ipc_port>]
hdfs dfsadmin [-fetchImage <local directory>]
hdfs dfsadmin [-allowSnapshot <snapshotDir>]
hdfs dfsadmin [-disallowSnapshot <snapshotDir>]
hdfs dfsadmin [-shutdownDatanode <datanode_host:ipc_port> [upgrade]]
hdfs dfsadmin [-evictWriters <datanode_host:ipc_port>]
hdfs dfsadmin [-getDatanodeInfo <datanode_host:ipc_port>]
hdfs dfsadmin [-metasave filename]
hdfs dfsadmin [-triggerBlockReport [-incremental] <datanode_host:ipc_port>]
hdfs dfsadmin [-listOpenFiles [-blockingDecommission] [-path <path>]]
hdfs dfsadmin [-help [cmd]]
```

| COMMAND_OPTION                                               | Description                                                  |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| `-report` `[-live]` `[-dead]` `[-decommissioning]` `[-enteringmaintenance]` `[-inmaintenance]` | 报告基本的文件系统信息和统计信息，dfs使用情况可能不同于“du”使用情况，因为它测量所有datanodes上的复制、校验和、快照等使用的原始空间。可选标志可用于筛选显示的数据节点列表。 |
| `-safemode` enter\|leave\|get\|wait\|forceExit               | 安全模式维护命令。安全模式是Namenode一个状态，安全模式下：1.不接受对名称空间更改(只读)；2. 不复制或删除块。安全模式在Namenode启动时自动进入，当配置的最小块百分比满足最小复制条件时自动离开安全模式。如果Namenode检测到任何异常，则它将在安全模式下逗留，直到该问题得到解决。如果异常是故意操作的结果，那么管理员可以使用-safe mode forceExit退出安全模式。可能需要强制退出的情况有：1.Namenode元数据不一致。如果Namenode检测到元数据已修改，并且可能导致数据丢失，则Namenode将进入forceExit状态。此时，用户可以使用正确的元数据文件重新启动Namenode或forceExit(如果可以接受数据丢失)。2. 回滚导致元数据被替换，并且很少能在Namenode中触发安全模式的forceExit状态。在这种情况下，可以通过发出-safemode forceExit来继续。 |
| `-saveNamespace` `[-beforeShutdown]`                         | 将当前命名空间保存到存储目录并重置编辑日志。需要安全模式。如果给定了“beforeShutdown”选项，则NameNode仅在时间窗口（检查点周期的可配置数量）内未完成检查点时才执行检查点。这通常在关闭NameNode之前使用，以防止潜在的fsimage/editlog损坏 |
| `-rollEdits`                                                 | 在活动的NameNode上手动执行编辑日志和fsimage合并              |
| `-restoreFailedStorage` true\|false\|check                   | 此选项将打开/关闭自动尝试恢复失败的存储副本。如果失败的存储再次可用，系统将在检查点期间尝试还原编辑和/或fsimage。“check”选项将返回当前设置 |
| `-refreshNodes`                                              | 重新读取namenode设置的数据节点黑名单和白名单                 |
| `-setQuota` <quota> <dirname>…<dirname>                      | See [HDFS Quotas Guide](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsQuotaAdminGuide.html#Administrative_Commands) for the detail. |
| `-clrQuota` <dirname>…<dirname>                              | See [HDFS Quotas Guide](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsQuotaAdminGuide.html#Administrative_Commands) for the detail. |
| `-setSpaceQuota` <quota> `[-storageType ]` <dirname>…<dirname> | See [HDFS Quotas Guide](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsQuotaAdminGuide.html#Administrative_Commands) for the detail. |
| `-clrSpaceQuota` `[-storageType ]` <dirname>…<dirname>       | See [HDFS Quotas Guide](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsQuotaAdminGuide.html#Administrative_Commands) for the detail. |
| `-finalizeUpgrade`                                           | 完成HDFS的升级。Datanodes删除它们以前版本的工作目录，然后Namenode执行相同的操作。这就完成了升级过程。 |
| `-rollingUpgrade` [<query>\|<prepare>\|<finalize>]           | See [Rolling Upgrade document](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsRollingUpgrade.html#dfsadmin_-rollingUpgrade) for the detail. |
| `-upgrade` query\|finalize                                   | 查询当前升级状态。完成HDFS的升级（相当于-Finalize upgrade）  |
| `-refreshServiceAcl`                                         | 重新加载服务级别授权策略文件。                               |
| `-refreshUserToGroupsMappings`                               | 刷新用户到组的映射                                           |
| `-refreshSuperUserGroupsConfiguration`                       | 刷新超级用户代理组映射                                       |
| `-refreshCallQueue`                                          | 从配置重新加载调用队列                                       |
| `-refresh` <host:ipc_port> <key> [arg1..argn]                | Triggers a runtime-refresh of the resource specified by <key> on <host:ipc_port>. All other args after are sent to the host. |
| `-reconfig` <datanode \|namenode> <host:ipc_port> <start\|status\|properties> | Starts reconfiguration or gets the status of an ongoing reconfiguration, or gets a list of reconfigurable properties. The second parameter specifies the node type. |
| `-printTopology`                                             | Print a tree of the racks and their nodes as reported by the Namenode |
| `-refreshNamenodes` datanodehost:port                        | For the given datanode, reloads the configuration files, stops serving the removed block-pools and starts serving new block-pools. |
| `-getVolumeReport` datanodehost:port                         | For the given datanode, get the volume report.               |
| `-deleteBlockPool` datanode-host:port blockpoolId [force]    | If force is passed, block pool directory for the given blockpool id on the given datanode is deleted along with its contents, otherwise the directory is deleted only if it is empty. The command will fail if datanode is still serving the block pool. Refer to refreshNamenodes to shutdown a block pool service on a datanode. |
| `-setBalancerBandwidth` <bandwidth in bytes per second>      | Changes the network bandwidth used by each datanode during HDFS block balancing. <bandwidth> is the maximum number of bytes per second that will be used by each datanode. This value overrides the dfs.datanode.balance.bandwidthPerSec parameter. NOTE: The new value is not persistent on the DataNode. |
| `-getBalancerBandwidth` <datanode_host:ipc_port>             | Get the network bandwidth(in bytes per second) for the given datanode. This is the maximum network bandwidth used by the datanode during HDFS block balancing. |
| `-fetchImage` <local directory>                              | Downloads the most recent fsimage from the NameNode and saves it in the specified local directory. |
| `-allowSnapshot` <snapshotDir>                               | Allowing snapshots of a directory to be created. If the operation completes successfully, the directory becomes snapshottable. See the [HDFS Snapshot Documentation](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsSnapshots.html) for more information. |
| `-disallowSnapshot` <snapshotDir>                            | Disallowing snapshots of a directory to be created. All snapshots of the directory must be deleted before disallowing snapshots. See the [HDFS Snapshot Documentation](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsSnapshots.html) for more information. |
| `-shutdownDatanode` <datanode_host:ipc_port> [upgrade]       | Submit a shutdown request for the given datanode. See [Rolling Upgrade document](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsRollingUpgrade.html#dfsadmin_-shutdownDatanode) for the detail. |
| `-evictWriters` <datanode_host:ipc_port>                     | Make the datanode evict all clients that are writing a block. This is useful if decommissioning is hung due to slow writers. |
| `-getDatanodeInfo` <datanode_host:ipc_port>                  | Get the information about the given datanode. See [Rolling Upgrade document](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsRollingUpgrade.html#dfsadmin_-getDatanodeInfo) for the detail. |
| `-metasave` filename                                         | Save Namenode’s primary data structures to *filename* in the directory specified by hadoop.log.dir property. *filename* is overwritten if it exists. *filename* will contain one line for each of the following 1. Datanodes heart beating with Namenode 2. Blocks waiting to be replicated 3. Blocks currently being replicated 4. Blocks waiting to be deleted |
| `-triggerBlockReport` `[-incremental]` <datanode_host:ipc_port> | Trigger a block report for the given datanode. If ‘incremental’ is specified, it will be otherwise, it will be a full block report. |
| `-listOpenFiles` `[-blockingDecommission]` `[-path ]`        | List all open files currently managed by the NameNode along with client name and client machine accessing them. Open files list will be filtered by given type and path. Add -blockingDecommission option if you only want to list open files that are blocking the DataNode decommissioning. |
| `-help` [cmd]                                                | Displays help for the given command or all commands if none is specified. |

