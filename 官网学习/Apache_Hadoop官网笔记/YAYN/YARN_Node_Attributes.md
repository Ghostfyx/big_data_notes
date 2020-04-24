# YARN Node Attributes

## 1. 概述

节点属性是一种无需资源保证即可描述一个节点的属性的方法。 应用程序可以使用大量这些属性的表达，来为要放置其容器的容器选择正确的节点。

## 2. 特性

节点属性的突出特性如下：

- 一个节点可以与多个属性关联。
- 值可以与标记到节点的属性关联，目前仅支持字符串类型值。
- 与节点标签不同，节点属性无需在集群级别明确指定，但是有API列出了集群级别可用的属性。
- 作为其无形资源，它不与任何队列关联，因此属性不需要队列资源规划和授权。
- 与分配标签类似，应用程序将能够使用包含放置约束的表达式来请求容器，这些表达式包含一个或多个这些属性。
- 表达式中仅支持等于(=)和不等于(！=)。 AND＆OR也可以用作属性表达式的一部分。
- 节点属性约束是硬性限制，也就是说只有在节点满足节点属性约束时才能进行分配。 换句话说，请求一直待处理，直到找到满足约束条件的有效节点为止。 目前没有放松政策
- 可操作性
	- 节点属性及其到节点的映射可以在RM重新启动时恢复
	- 更新节点属性-RM运行时，管理员可以在节点上添加，删除和替换属性
- NM到节点属性的映射可以通过两种方式完成：
	- **Centralised** 可以通过RM公开的CLI或RPC(尚未支持REST)完成节点到属性的映射
	- **Distributed** 节点到属性的映射将由NM中配置的节点属性提供程序设置。 YARN中有两个不同的提供程序：基于脚本的提供程序和基于配置的提供程序。 如果是脚本，则可以使用脚本路径配置NM，并且脚本可以发出节点的属性。 如果是配置，则可以在NM的yarn-site.xml中直接配置节点属性。 这两个选项均支持属性映射的动态刷新。
- 与标签不同，属性可以同时从集中式模式和分布式模式映射到节点。 由于在不同模式下使用不同的前缀来标识属性，因此不会发生冲突。 如果是集中式属性，则用前缀“ rm.yarn.io”标识；如果是分布式，则属性用前缀“ nm.yarn.io”标识。 这意味着属性由前缀和名称唯一标识。

## 3. 配置

### 3.1 ResourceManager设置节点属性

与标签不同，可以使用以下两种方式，通过为节点属性设置ResourceManager将属性映射到节点：

在yarn-site.xml设置如下属性：

| Property                                | Value                                                        | Default Value                                                |
| :-------------------------------------- | :----------------------------------------------------------- | :----------------------------------------------------------- |
| yarn.node-attribute.fs-store.root-dir   | 集中式属性映射的存储路径                                     | [file:///tmp/hadoop-yarn-${user}/node-attribute/](file:///tmp/hadoop-yarn-${user}/node-attribute/) |
| yarn.node-attribute.fs-store.impl.class | 配置的类需要扩展org.apache.hadoop.yarn.nodelabels.NodeAttributeStore | FileSystemNodeAttributeStore                                 |

注意：

- 确保使用资源管理器进程用户(通常是来自YARN用户)创建yarn.node-attribute.fs-store.root-dir，并且ResourceManager有权访问它。
- 如果用户要将节点属性存储到RM的本地文件系统，则可以使用诸如：file:///home/yarn/node-attributes之类的路径，或者使用hdfs存储，可以使用路径如：hdfs://namenode:port/path/to/store/node-attributes/ 

### 3.2 集中式节点属性映射

在集中模式中，支持三个选项将属性映射到节点：

- 添加命令：`yarn nodeattributes -add “node1:attribute[(type)][=value],attribute2 node2:attribute2[=value],attribute3` 将属性添加到节点，而不会影响节点上已经存在的映射
- 移除属性命令：`yarn nodeattributes -remove “node1:attribute,attribute1 node2:attribute2`删除节点的属性，而不会影响节点上已经存在的映射
- 替换命令：`yarn nodeattributes -replace “node1:attribute[(type)][=value],attribute1[=value],attribute2 node2:attribute2[=value],attribute3`使用此命令中配置的属性替换节点的现有属性

注意：

- 无需使用端口，属性映射到节点中的所有NM实例
- 命令空间是多个节点-属性映射对的定界符
- “，”用作节点的多个属性的定界符
- 如果未指定，则“ type”默认为字符串，这是当前唯一支持的类型。
- 以上3种操作只能由admin用户执行

### 3.2 分布式节点属性映射

在分布式模式下配置节点的属性

| Property                                                     | Value                                                        |
| :----------------------------------------------------------- | :----------------------------------------------------------- |
| yarn.nodemanager.node-attributes.provider                    | 管理员可以通过在NM中配置此参数来配置节点属性的provider。 管理员为provider配置 config， script或类名。配置的类需要扩展org.apache.hadoop.yarn.server.nodemanager.nodelabels.NodeAttributesProvider。 如果配置config，则使用ConfigurationNodeAttributesProvider，如果配置了“ script”，Script则将使用“ScriptBasedNodeAttributesProvider。 |
| yarn.nodemanager.node-attributes.provider.fetch-interval-ms  | 当使用config，script或configured clas扩展了yarn.nodemanager.node-attributes.provider时，会定期从节点属性提供者中检索节点属性。 此配置用于定义间隔时间。 如果配置了-1，则仅在初始化期间从提供程序中检索节点属性。 默认为10分钟。 |
| yarn.nodemanager.node-attributes.provider.fetch-timeout-ms   | 当用脚本配置yarn.nodemanager.node-attributes.provider时，此配置将提供超时时间，在此之后它将中断查询节点属性的脚本。 默认为20分钟。 |
| yarn.nodemanager.node-attributes.provider.script.path        | 节点属性脚本NM运行以收集节点属性。 脚本输出中以NODE_ATTRIBUTE：开头的行将被视为节点属性的记录，属性名称，类型和值应以逗号分隔。 每个这样的行将被解析为一个节点属性。 |
| yarn.nodemanager.node-attributes.provider.script.opts        | 传递给节点属性脚本的参数。                                   |
| yarn.nodemanager.node-attributes.provider.configured-node-attributes | 当用config配置yarn.nodemanager.node-attributes.provider时，ConfigurationNodeAttributesProvider从该参数获取节点属性。 |

## 4. 应用程序指定节点属性

应用程序可以使用Placement Constraint API来指定节点属性请求，如Placement Constraint文档中所述。

以下是使用NodeAttribute表达式创建Scheduling Request对象的示例：

```java
//expression : AND(python!=3:java=1.8)
SchedulingRequest schedulingRequest =
    SchedulingRequest.newBuilder().executionType(
        ExecutionTypeRequest.newInstance(ExecutionType.GUARANTEED))
        .allocationRequestId(10L).priority(Priority.newInstance(1))
        .placementConstraintExpression(
            PlacementConstraints.and(
                PlacementConstraints
                    .targetNodeAttribute(PlacementConstraints.NODE,
                        NodeAttributeOpCode.NE,
                        PlacementConstraints.PlacementTargets
                            .nodeAttribute("python", "3")),
                PlacementConstraints
                    .targetNodeAttribute(PlacementConstraints.NODE,
                        NodeAttributeOpCode.EQ,
                        PlacementConstraints.PlacementTargets
                            .nodeAttribute("java", "1.8")))
                .build()).resourceSizing(
        ResourceSizing.newInstance(1, Resource.newInstance(1024, 1)))
        .build();
```

上面的SchedulingRequest请求节点上的1个容器必须满足以下约束：

1. 节点属性rm.yarn.io/python在节点上不存在或存在，但其值不等于3
2. 节点属性rm.yarn.io/java必须存在于节点上，其值等于1.8

## 5. 监控

### 5.1 使用REST方式监控

作为`http://rm-http-address:port/ws/v1/cluster/nodes/{nodeid}`的一部分，可以获得REST输出属性及其映射到给定节点的值。

### 5.2 Web UI方式监控

当前不支持

### 5.3 命令行方式监控

- 使用`yarn cluster --list-node-attributes`获取集群中的所有属性
- 使用 `yarn nodeattributes -list`获取集群中的属性
- 使用`yarn nodeattributes -attributestonodes -attributes <Attributes>`列出每个属性，所有映射的节点以及为每个节点配置的属性值。可以使用-attributes指定的属性
- 使用`yarn nodeattributes -nodestoattributes -nodes <Host Names>`列出所有属性，以及映射到节点的值，使用-nodes为指定的节点指定
- 获取节点状态/详情的命令`yarn node -status`，将列出所有与节点关联的属性及其值

