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

- 在**Distributed** 节点类型下配置节点到标签的映射。

| Property                                                     | Value                                                        |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| yarn.node-labels.configuration-type                          | 需要在RM中设置为Distributed，从NM中已配置的节点标签获取节点到标签映射的映射 |
| yarn.nodemanager.node-labels.provider                        | 在RM中将yarn.node-labels.configuration-type配置为distributed时，管理员可以通过在NM中配置此参数来配置节点标签的提供程序。管理员将provider设置为config、script或类名。 配置的类需要扩展org.apache.hadoop.yarn.server.nodemanager.nodelabels.NodeLabelsProvid如果配置为config，则使用ConfigurationNodeLabelsProvider；如果配置为 script，则将使用ScriptNodeLabelsProvider |
| yarn.nodemanager.node-labels.resync-interval-ms              | NM与RM同步其节点标签的时间间隔。 NM将每配置x个间隔将其加载的标签以及心跳发送给RM。 即使未修改标签，也需要重新同步，因为admin可能已删除了NM提供的群集标签。 默认值为2分钟 |
| yarn.nodemanager.node-labels.provider.fetch-interval-ms      | 当使用config、script或扩展AbstractNodeLabelsProvider类来配置yarn.nodemanager.node-labels.provider时，则定期从节点标签提供者中检索节点标签。 此配置用于定义间隔时间。 如果配置了-1，则仅在初始化期间从提供程序中检索节点标签。 默认为10分钟 |
| yarn.nodemanager.node-labels.provider.fetch-timeout-ms       | 如果将yarn.nodemanager.node-labels.provider配置为script，则此配置将提供超时时间，在此之后它将中断查询节点标签的脚本。 默认为20分钟。 |
| yarn.nodemanager.node-labels.provider.script.path            | 要运行的节点标签脚本。 以NODE_PARTITION:开头的脚本输出行将被视为节点标签分区。 如果脚本输出的多行具有此模式，则将考虑最后一行。 |
| yarn.nodemanager.node-labels.provider.script.opts            | 传递给节点标签脚本的参数                                     |
| yarn.nodemanager.node-labels.provider.configured-node-partition | 当使用config配置yarn.nodemanager.node-labels.provider时，ConfigurationNodeLabelsProvider从该参数获取分区标签 |

- 在**Delegated-Centralized** 节点类型下配置节点到标签的映射

| Property                                                    | Value                                                        |
| :---------------------------------------------------------- | :----------------------------------------------------------- |
| yarn.node-labels.configuration-type                         | 需要设置为delegated-centralized，以从RM中已配置的Node Labels Provider获取节点到标签的映射 |
| yarn.resourcemanager.node-labels.provider                   | 如果将yarn.node-labels.configuration-type配置为delegated-centralized，则管理员应配置该类以通过ResourceManager提取节点标签。 配置的类需要扩展org.apache.hadoop.yarn.server.resourcemanager.nodelabels.RMNodeLabelsMappingProvider 。 |
| yarn.resourcemanager.node-labels.provider.fetch-interval-ms | 如果将 yarn.node-labels.configuration-type配置为delegated-centralized，则会定期从节点标签提供程序中检索节点标签。 此配置用于定义间隔。 如果配置了-1，则在每个节点注册后，仅从提供者中检索节点标签一次。 默认为30分钟。 |

### 3.5 节点标签调度器的配置

- 容量调度器配置

| Property                                                     | Value                                                        |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| yarn.scheduler.capacity.< queue-path>.capacity               | 设置可以访问属于DEFAULT分区节点的队列百分比。 每个父母下直系子女的默认能力总和必须等于100。 |
| yarn.scheduler.capacity.< queue-path>.accessible-node-labels | 管理员需要指定每个队列可以访问的标签，并用逗号分隔，例如“ hbase，storm”表示队列可以访问标签hbase和storm。所有队列都可以访问不带标签的节点，用户无需指定。 如果用户未指定此字段，它将从其父级继承。 如果用户要明确指定只能访问没有标签的节点的队列，只需将空格作为值即可。 |
| yarn.scheduler.capacity.< queue-path>.accessible-node-labels.<label>.capacity | 设置队列可以访问属于< label>分区的节点的百分比。 每个父母下直系子女的< label>分区容量总和必须等于100。默认情况下为0。 |
| yarn.scheduler.capacity.< queue-path>.accessible-node-labels.<label>.maximum-capacity | 类似于yarn.scheduler.capacity.<queue-path>.maximum-capacity，用于每个队列的标签的最大容量。 默认情况下为100 |
| yarn.scheduler.capacity.< queue-path>.default-node-label-expression | 假设默认节点标签为hbase，则表示：如果应用程序提交到队列时未在资源请求中指定节点标签，则它将使用hbase作为default-node-label-expression。 默认情况下，它为空，因此应用程序将从没有标签的节点获取容器。 |

节点标签配置示例：

假设集群有以下队列结构

```
  							root
            /     |    \
     engineer    sales  marketing
```

集群中有5个节点(主机名= h1..h5)，每个节点都有24G内存和24个vcore。 5个节点中有1个具有GPU(假设是h5)。 因此，管理员在h5中添加了GPU标签。

假设用户具有如下的Capacity Scheduler配置(在此处使用key = value以提高可读性)：

```xml
yarn.scheduler.capacity.root.queues=engineering,marketing,sales
yarn.scheduler.capacity.root.engineering.capacity=33
yarn.scheduler.capacity.root.marketing.capacity=34
yarn.scheduler.capacity.root.sales.capacity=33

yarn.scheduler.capacity.root.engineering.accessible-node-labels=GPU
yarn.scheduler.capacity.root.marketing.accessible-node-labels=GPU

yarn.scheduler.capacity.root.engineering.accessible-node-labels.GPU.capacity=50
yarn.scheduler.capacity.root.marketing.accessible-node-labels.GPU.capacity=50

yarn.scheduler.capacity.root.engineering.default-node-label-expression=GPU
```

可以看到root.engineering/marketing/sales.capacity=33，因此每个子队列的资源都可以保证等于总资源的1/3且没有分区。 因此每个子队列都可以使用h1..h4的1/3资源，即$24*4*\frac{1}{3}=32G,32vcores$。

而且只有engineering/marketing队列有权访问GPU分区(请参阅 root.<queue-name>.accessible-node-labels)。

每个engineering/marketing队列均已保证资源等于partition=GPU的资源的1/2。 因此他们每个人都可以使用h5的1/2资源，即$24 * 0.5 =12G mem，12vcores$。

注意：

- 完成CapacityScheduler的配置后，执行yarn rmadmin -refreshQueues以应用配置更改
- 进入RM Web UI的调度页面，查看配置是否成功

## 4. 制定应用的节点标签

应用程序可以使用以下Java API来指定要请求的节点标签：

- `ApplicationSubmissionContext.setNodeLabelExpression(..)`为应用程序的所有容器设置节点标签表达式
- `ResourceRequest.setNodeLabelExpression(..)`为单个资源请求设置节点标签表达式。 这可以覆盖ApplicationSubmissionContext中设置的节点标签表达式
- 在ApplicationSubmissionContext中设置`setAMContainerResourceRequest.setNodeLabelExpression`以指定应用程序主容器的预期节点标签

## 5. 监控

### 5.1 Web UI监控

在Web UI上可以看到以下与标签相关的字段：

- Nodes page: [http://RM-Address:port/cluster/nodes](http://rm-address:port/cluster/nodes), 可以获取每个节点的节点标签
- Node labels page: [http://RM-Address:port/cluster/nodelabels](http://rm-address:port/cluster/nodelabels)，可以获取类型(exclusive/non-exclusive)，活动节点管理器的数量，每个分区的总资源
- Scheduler page: [http://RM-Address:port/cluster/scheduler](http://rm-address:port/cluster/scheduler)，可以获得每个队列的标签相关设置以及队列分区的资源使用情况

### 5.2 命令行监控

- 使用`yarn cluster --list-node-labe`命令获取集群所有节点标签
- 使用`yarn node -status <NodeId>`命令获取节点状态，包括给定节点上的标签