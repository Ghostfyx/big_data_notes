# ResourceManager Restart

 ## 1. 概述

ResourceManager是运行在YARN上，负责管理资源和调度应用程序的核心组件。因此，它可能是Apache YARN群集中的单点故障。 本文档概述了ResourceManager重新启动，该特性增强了ResourceManager在重启过程中保持集群可用，使ResourceManager停机期间对用户不可见。

下面是ResourceManageger两种重启方式：

- **Non-work-preserving RM restart**  这种重启方式通过将应用程序/推测执行状态与其他的凭据信息保留在可插拔状态存储中。 RM将在重新启动时从状态存储重新加载此信息，并重新启动以前运行的应用程序，不需要用户重新提交应用程序。PS：此方式Hadoop 2.4.0版本实现。
- **Work-preserving RM restart**  通过结合NodeManagers的容器状态和重新启动时来自ApplicationMasters的容器请求来重建RM的运行状态。与不保留工作的RM重新启动的主要区别在于，RM重新启动后，先前运行的应用程序不会被杀死，因此应用程序不会因为RM中断而丢失工作。PS：此方式在Hadoop 2.6.0版本实现。

## 2. 特性

### 2.1 Non-work-preserving RM restart

不保留工作的RM重新启动时，RM将客户端提交应用时的元数据(即ApplicationSubmissionContext)保存在可插拔状态存储中，同时保存其最终状态(例如完成状态、失败、中止或终止)，并在应用程序完成时进行诊断。此外，如果在安全环境中工作，RM还保存凭据(如安全密钥，令牌)。

当RM关闭时，只要状态存储中提供了所需的信息，即应用程序元数据和在安全环境中运行的凭据，则RM重新启动时，它可以从状态库中获取应用程序元数据，存储并重新提交应用程序。如果RM在停机之前已经完成(即失败，中止或完成)，则它们不会重新提交申请。

在RM停机期间，NodeManager和客户端将继续轮询RM，直到RM重新相应为止。 RM启动时，它将通过心跳向正在与之通信的所有NodeManager和ApplicationMaster发送重新同步命令。NM将杀死其所有托管容器，并向RM重新注册，这些重新注册的NodeManager与新加入的NM相似。

AMs(例如：MapReduce的ApplicationMaster)在收到重新同步命令后，将会杀掉自己。RM重新启动并从状态存储加载所有应用程序元数据、凭据并将其写入到内存后，它将为每个未完成的应用新建ApplicationMaster，并照常重新运行该应用。 如前所述，以前运行的应用以这种方式丢失，因为它们实际上是在运行通过re-sync命令被RM杀死的(其运行的中间数据将会丢失)。

### 2.2 **Work-preserving RM restart**

在工作保留RM重启中，RM确保应用程序状态的持久化并在恢复时重新加载该状态，此重启主要侧重于重构YARN集群的整个运行状态，其中大部分是RM内部调度程序的状态。它跟踪所有容器的生命周期，余量，资源请求和队列的资源使用情况等。通过这种方式，RM无需终止AM并从头开始重新运行应用程序，因为它是在非工作保留的RM重启中完成的。应用程序可以简单地与RM重新同步，并从中断处继续。

RM利用从所有NM发送的容器状态来恢复其运行状态。当NM与重新启动的RM重新同步时，NM不会杀死容器。它继续管理容器，并在重新注册时将容器状态发送到RM。RM通过吸收这些容器的信息来重建容器实例和相关应用程序的调度状态。与此同时，AM需要将未完成的资源请求重新发送给RM，因为RM可能会在关闭时丢失未完成的请求。使用AMRMClient库与RM通信的应用程序编写者无需担心AM在重新同步时向RM重新发送资源请求的部分，因为它自动由库本身处理。

## 3. 配置

本节介绍了启用RM重新启动功能所涉及的配置。

### 3.1 **启用RM重启**

| 属性                                  | 描述 |
| :------------------------------------ | :--- |
| yarn.resourcemanager.recovery.enabled | true |

### 3.2 配置状态存储以保存RM状态

| 属性                             | 描述                                        |
| :------------------------------- | :------------------------------------------ |
| yarn.resourcemanager.store.class | 用于保存应用/尝试状态和凭据的状态存储的类名 |

可用的状态存储如下：

- 基于ZooKeeper：ZKRMStateStore
- 基于Hadoop文件系统，例如HDFS和本地FS：FileSystemRMStateStore
- 基于LevelDB：LeveldbRMStateStore

默认值为org.apache.hadoop.yarn.server.resourcemanager.recovery.FileSystemRMStateStore。

### 3.3 如何选择状态存储类型

- **基于ZooKeeper的状态存储**：用户可以自由选择任何存储来设置RM重启，但必须使用基于ZooKeeper的状态存储来支持RM HA。原因是只有基于ZooKeeper的状态存储支持防护机制，以避免多个RM假设它们处于活动状态并且可以同时编辑状态存储的裂脑情况。
- **基于文件系统的状态存储**：支持HDFS和基于本地FS的状态存储。不支持防护机制。
- **基于LevelDB的状态存储**：基于LevelDB的状态存储被认为比基于HDFS和ZooKeeper的状态存储更轻。LevelDB支持更好的原子操作，每个状态更新更少的I/O操作，以及文件系统上的总文件更少。不支持防护机制。

### 3.4 **基于Hadoop FileSystem的状态存储实现的配置**

文件系统的URI Schema决定使用HDFS还是本地FS作为状态存储实现，例如hdfs://localhost:9000/rmstore 使用HDFS作为存储和 file:///tmp/yarn/rmstore 使用本地FS作为存储。若URI中未指定Schema(hdfs://或file://)，则要使用的存储类型由core-site.xml中定义的fs.defaultFS决定。

- 配置将在Hadoop FileSystem状态存储中保存RM状态的URI

| 属性                                    | 描述                                                         |
| :-------------------------------------- | :----------------------------------------------------------- |
| yarn.resourcemanager.fs.state-store.uri | 存储RM状态的FileSystem路径，例如hdfs://localhost:9000/rmstore。默认值为{hadoop.tmp.dir}/yarn/system/rmstore 。如果未提供FileSystem URI，则将使用core-site.xml中指定的fs.default.name。 |

- 配置状态存储客户端用于连接Hadoop FileSystem的重试策略

| 属性                                                  | 描述                                                         |
| :---------------------------------------------------- | :----------------------------------------------------------- |
| yarn.resourcemanager.fs.state-store.retry-policy-spec | Hadoop FileSystem客户端重试策略规范。始终启用Hadoop FileSystem客户端重试。设置格式 (睡眠时间,重试次数)，即$(t_0，n_0),(t_1,n_1), \dots$第一个$n_0$重试平均休眠$t_0$毫秒，以下$n_1$重试平均休眠$t_1$毫秒。默认值为(2000,500) |

 ### 3.5 基于ZooKeeper的状态存储实现的配置

- 配置ZooKeeper服务器地址和存储RM状态的根路径

| 属性                                            | 描述                                                         |
| :---------------------------------------------- | :----------------------------------------------------------- |
| hadoop.zk.address                               | 以逗号分隔的主机列表:端口对。每个对应用于存储RM状态的ZooKeeper服务器(例如`127.0.0.1:3000,127.0.0.1:3001,127.0.0.1:3002`) |
| yarn.resourcemanager.zk-state-store.parent-path | 存储RM状态的根znode的完整路径，默认值为 /rmstore             |

- 配置状态存储客户端用于连接ZooKeeper服务器的重试策略

| 属性                        | 描述                                                         |
| :-------------------------- | :----------------------------------------------------------- |
| hadoop.zk.num-retries       | 如果连接丢失，RM尝试连接到ZooKeeper服务器的次数。默认值为500 |
| hadoop.zk.retry-interval-ms | 连接到ZooKeeper服务器时重试之间的间隔(以毫秒为单位)。默认值为2000毫秒 |
| hadoop.zk.timeout-ms        | ZooKeeper会话超时时间(以毫秒为单位)。ZooKeeper服务器使用此配置来确定会话何时到期。当服务器在此配置指定的会话超时期限内没有从客户端收到消息(即没有心跳)时，会发生会话到期。默认10000毫秒 |

- 配置用于在ZooKeeper znodes上设置权限的ACL

| 属性          | 描述                                                         |
| :------------ | :----------------------------------------------------------- |
| hadoop.zk.acl | 用于在ZooKeeper znodes上设置权限的ACL。默认值为world:everyone:rwcda |

### 3.6 基于LevelDB的状态存储实现的配置

| 属性                                          | 描述                                                         |
| :-------------------------------------------- | :----------------------------------------------------------- |
| yarn.resourcemanager.leveldb-state-store.path | 将存储RM状态的levelDB路径。默认值 {hadoop.tmp.dir}/yarn/system/rmstore |

### 3.7 保持工作的RM恢复配置

| 属性                                                         | 描述                                                         |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| yarn.resourcemanager.work-preserving-recovery.scheduling-wait-ms | 设置在RM中保留的工作恢复分配新容器之前，RM等待的时间。 在为应用程序分配新容器之前，这样的等待时间使RM有机会在恢复时解决与群集中NM的重新同步 |

## 4. 笔记

如果RM重新启动并启用了工作保留恢复，则会更改ContainerId字符串格式。它格式如下：Container_ {clusterTimestamp} _ {appId} _ {attemptId} _ {containerId}，例如Container_1410901177871_0001_01_000005。

它现在更改为：Container_e **{epoch}** _ {clusterTimestamp} _ {appId} _ {attemptId} _ {containerId}，例如Container_ **e17** _1410901177871_0001_01_000005。

这里，附加纪元号是单调递增的整数，从0开始并且每次RM重新启动时增加1。如果纪元号为0，则省略它，containerId字符串格式保持与以前相同。

## 5. 简单配置样例

以下是启用RM工作保留，状态存储使用基于ZooKeeper的状态存储的最低配置集。

```xml
<property>
   <description>Enable RM to recover state after starting. If true, then
   yarn.resourcemanager.store.class must be specified</description>
   <name>yarn.resourcemanager.recovery.enabled</name>
   <value>true</value>
 </property>
 
 <property>
   <description>The class to use as the persistent store.</description>
   <name>yarn.resourcemanager.store.class</name>
   <value>org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore</value>
 </property>
 
 <property>
   <description>Comma separated list of Host:Port pairs. Each corresponds to a ZooKeeper server
   (e.g. "127.0.0.1:3000,127.0.0.1:3001,127.0.0.1:3002") to be used by the RM for storing RM state.
   This must be supplied when using org.apache.hadoop.yarn.server.resourcemanager.recovery.ZKRMStateStore
   as the value for yarn.resourcemanager.store.class</description>
   <name>hadoop.zk.address</name>
   <value>127.0.0.1:2181</value>
 </property>
```

