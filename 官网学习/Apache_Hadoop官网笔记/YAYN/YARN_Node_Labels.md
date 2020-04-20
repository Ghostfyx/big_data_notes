# YARN Node Labels

 ## 1. 概述

节点标签是对具有相似特征的节点进行分组的一种方法，应用程序可以指定任意节点类型上运行。

现在仅支持节点分区，即：

- 一个节点只能有一个分区，因此，群集按节点分区划分为几个不相交的子群集。 默认情况下，节点属于DEFAULT分区(partition =“ ”)
- 用户需要配置每个分区可以由不同队列使用多少资源。 
- 有两种节点分区：
	- Exclusive：容器将分配给节点分区完全匹配的节点(例如，将请求分区partition =“ x”分配给具有partition =“ x”的节点，将请求分区DEFAULT分区分配给DEFAULT分区节点)。
	- Non-exclusive：如果分区是非排他性的，则它将空闲资源共享给请求DEFAULT分区的容器。

用户可以指定每个队列可以访问的一组节点标签，一个应用程序只能使用包含其应用队列可以访问的节点标签的子集。

## 2. 特性

当前节点标签支持一下特性：

- Partition cluster 每个节点可以分配一个标签，因此群集将被划分为几个较小的不相交的分区。
- ACL of node-labels on queues 用户可以设置每个队列可使用的节点分区，因此只有某些节点只能被特定的队列访问。
- 指定可以由队列访问的分区资源的百分比  用户可以设置百分比，例如：队列A可以访问具有label = hbase的节点上30％的资源。 这样的百分比设置将与现有资源管理器一致
- 在请求资源时制定节点标签，仅当节点具有相同标签时才分配它。 如果未指定节点标签要求，则仅在属于DEFAULT分区的节点上分配这种资源请求。 
- 可操作性
	- 可以在RM重新启动期间恢复节点标签和节点标签映射
	- 更新节点标签-RM运行时，管理员可以更新节点上标签和队列上的标签
- NodeManager到节点标签的映射可以用三种方式配置，但是在所有方法中，Partition Label应该是RM中配置的有效节点标签列表之一。
	- **Centralized** 节点到标签的映射可以通过RM公开的CLI，REST或RPC完成。
	- **Distributed**   节点到标签的映射将由NM中配置的节点标签提供设置。YARN中有两个不同的设置方式：基于脚本的设置和基于配置的设置。使用脚本时，可以使用脚本路径配置NM，并且脚本可以发出节点的标签。 基于配置时，则可以在NM的yarn-site.xml中直接配置节点标签。 这两个配置均支持标签映射的动态刷新。
	- **Delegated-Centralized**  节点到标签的映射将由RM中配置的节点标签提供设置。 当由于安全问题而无法由每个节点提供标签映射时，这将很有帮助，并且可以避免大型集群中每个节点通过RM接口进行交互。 在NM注册期间将从该接口获取标签，并且还支持定期刷新。

## 3. 配置

### 3.1 设置ResourceManager启动节点标签

在yarn-site.xml设置以下属性：

| Property                            | Value                                                        |
| :---------------------------------- | :----------------------------------------------------------- |
| yarn.node-labels.fs-store.root-dir  | [hdfs://namenode:port/path/to/store/node-labels/](hdfs://namenode:port/path/to/store/node-labels/) |
| yarn.node-labels.enabled            | true                                                         |
| yarn.node-labels.configuration-type | 设置节点标签的配置类型。 管理员可以指定centralized、delegated-centralized或distributed，默认为centralized |

注意：

- 确保已创建yarn.node-labels.fs-store.root-dir，并且ResourceManager有权访问它(通常由 yarn用户创建)。
- 如果用户要将节点标签存储到RM的本地文件系统（而不是HDFS），则可以使用file:///home/yarn/node-label这样的路径

### 3.2 新增/修改YARN中的节点类型列表

新增节点类型列表：

- 执行`yarn rmadmin -addToClusterNodeLabels label_1(exclusive=true/false),label_2(...)`添加节点标签。
- 如果用户未指定exclusive，exclusive默认为true。
- 运行`yarn cluster --list-node-labels`检查添加新增的节点类型可见

### 3.3 移除YARN中的节点类型

移除节点类型：

- `yarn rmadmin -removeFromClusterNodeLabels "<label>[,<label>,...]"`命令移除一个或多个节点标签，
- 不允许删除与队列相关联的标签，即一个或多个队列可以访问此标签。
- 要验证是否已成功删除指定的节点标签，请运行`yarn cluster --list-node-labels`。

### 3.4 新增/修改节点到标签的映射到YARN

在Centralized模式下配置节点到标签的映射。

- 执行`yarn rmadmin -replaceLabelsOnNode “node1[:port]=label1 node2=label2” [-failOnUnknownNodes]`。将label1添加到node1，将label2添加到node2。 如果用户未指定端口，则会将标签添加到节点上运行的所有NodeManager。 如果设置了选项-failOnUnknownNodes，则如果指定的节点未知，该命令将失败。

在**Distributed** 节点类型下配置节点到标签的映射。

| Property                            | Value |
| :---------------------------------- | :---- |
| yarn.node-labels.configuration-type |       |