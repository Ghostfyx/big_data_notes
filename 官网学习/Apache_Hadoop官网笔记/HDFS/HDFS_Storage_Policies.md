# Archival Storage, SSD & Memory

 ## 1. 简介

分级存储(Archival Storage)是一种可将不断增长的存储容量与计算容量分离的解决方案。具有更高密度和价格便宜，计算能力低的存储节点正变得可用，并且可以用作群集中的冷存储。根据策略，热数据可以移至冷数据。 向冷存储器中添加更多节点可以独立于群集中的计算容量而增加存储量。

异构存储和分级存储提供的框架将HDFS架构扩展为包括其他类型的存储介质，包括SSD和内存。用户可以选择将数据存储在SSD或内存中以获得更好的性能。

## 2. 存储类型与存储策略

### 2.1 存储类型：ARCHIVE, DISK, SSD与RAM_DISK

异构存储的第一阶段将datanode存储模型从单个存储(可能对应于多个物理存储介质)更改为存储集合，每个存储都对应于一个物理存储介质。 它还添加了存储类型DISK和SSD的概念，其中DISK是默认存储类型。

新增了一种新的存储类型ARCHIVE，它具有很高的存储密度(PB级存储)，但计算能力却很小，可以支持归档存储。

新增另一新的存储类型RAM_DISK，以支持在内存中写入单个副本文件。

### 2.2 存储策略：Hot, Warm, Cold, All_SSD, One_SSD, Lazy_Persist and Provided

引入了新概念——存储策略，允许根据存储策略将文件存储在不同的存储类型中。

有以下存储策略：

- **hot**：用于存储和计算。 常用且仍用于任务重的数据将保留在此策略中。 当一个块经常使用时，所有副本都存储在DISK中
- **Cold**：仅用于有限计算的存储。 不再使用的数据或需要归档的数据从热存储移动到冷存储。 当块处于冷状态时，所有副本都存储在ARCHIVE中。
- **Warm**：数据处理冷/热之间。当块处于warm状态，某些副本存储在DISK中，其余副本存储在ARCHIVE中。
- **All_SSD**：所有复本存储在SSD上
- **One_SSD**：其中一个副本在SSD上，其余副本存储在DISK中
- **Lazy_Persist**：用于在内存中写入具有单个副本的块。 首先将副本写入RAM_DISK，然后将其延迟保存在DISK中
- **Provided**：用于在HDFS外部存储数据，参阅[HDFS Provided Storage](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsProvidedStorage.html)

存储策略由以下字段组成：

1. policy ID

2. policy Name

3. 块的存储类型列表

4. 文件创建的后备存储类型列表

5. 文件副本的后备存储类型列表

当有足够的空间时，将根据3中指定的存储类型列表存储块副本。 当列表3中的某些存储类型空间不足时，将分别使用4和5中指定的后备存储类型列表替换空间不足的存储类型，用于文件创建和复制。

以下是典型的存储策略表：

| **Policy** **ID** | **Policy** **Name** | **Block Placement** **(n replicas)** | **Fallback storages** **for creation** | **Fallback storages** **for replication** |
| :---------------- | :------------------ | :----------------------------------- | :------------------------------------- | :---------------------------------------- |
| 15                | Lazy_Persist        | RAM_DISK: 1, DISK: n-1               | DISK                                   | DISK                                      |
| 12                | All_SSD             | SSD: n                               | DISK                                   | DISK                                      |
| 10                | One_SSD             | SSD: 1, DISK: n-1                    | SSD, DISK                              | SSD, DISK                                 |
| 7                 | Hot (default)       | DISK: n                              | <none>                                 | ARCHIVE                                   |
| 5                 | Warm                | DISK: 1, ARCHIVE: *n*-1              | ARCHIVE, DISK                          | ARCHIVE, DISK                             |
| 2                 | Cold                | ARCHIVE: n                           | <none>                                 | <none>                                    |
| 1                 | Provided            | PROVIDED: 1, DISK: n-1               | PROVIDED, DISK                         | PROVIDED, DISK                            |

注意1：Lazy_Persist策略仅对单个副本块有用。 对于具有多个副本的块，所有副本将被写入DISK，因为仅将其中一个副本写入RAM_DISK不会提高整体性能。

注2：对于具有条带化布局的纠错码文件，合适的存储策略为All_SSD，热，冷。 因此，如果用户为条带化EC文件设置了除上述策略以外的其他策略，则在创建或移动块时不会遵循该策略。

### 2.3 存储策略解析

创建文件或目录时，未指定其存储策略(使用storagepolicies -setStoragePolicy命令指定存储策略)。 文件或目录的有效存储策略由以下规则决定：

1. 如果文件或目录指定了存储策略，则返回它
2. 对于未指定的文件或目录，如果它是根目录，返回默认存储策略。 否则，返回其父目录的有效存储策略

可以通过`storagepolicies -getStoragePolicy`命令检索有效的存储策略。

### 2.4 配置

- **dfs.storage.policy.enabled** 用于启用/禁用存储策略，默认值是true(开启)。
- **dfs.datanode.data.dir** 在每个数据节点上，应使用逗号分隔的存储位置标记其存储类型。 这允许存储策略根据策略将块放置在不同的存储类型上。 例如：
	1. 数据节点存储位置`/grid/dn/disk0`在DISK上应该被配置为` [DISK]file:///grid/dn/disk0`
	2. 数据节点存储位置`/grid/dn/disk0`在SSD上应该被配置为`[SSD]file:///grid/dn/ssd0`
	3. 数据节点存储位置`/grid/dn/archive0`在ARCHIVE上应该被配置为`[ARCHIVE]file:///grid/dn/archive0`
	4. 数据节点存储位置`/grid/dn/ram0`在RAM_DISK上应该被配置为`[RAM_DISK]file:///grid/dn/ram0`

如果没有显式标记的数据节点存储位置，其默认存储类型为DISK。

## 3. 基于存储策略的数据移动

在已经存在的文件/目录上设置新的存储策略将更改命名空间中的策略，但不会在存储介质之间物理移动块。 以下2个选项将允许用户根据新策略集移动块。因此，一旦用户更改或设置了文件/目录上的新策略，用户还应执行以下选项之一以实现所需的数据移动。 注意，两个选项不能同时运行。

### 3.1 Storage Policy Satisfier (SPS)

当用户更改文件/目录上的存储策略时，用户可以调用HdfsAdmin API `StoragePolicy()`以按照新策略集移动块。 在namenode外部运行的SPS工具会定期扫描并发现新策略集和放置的物理块之间的存储不匹配。 将会尽对用户调用 `StoragePolicy()`的文件。如果SPS标记了文件要移动的块，会将调度块移动任务到数据节点。

SPS可以作为Namenode外部的外部服务启用，也可以动态禁用，无需重新启动Namenode。

可以参考[Storage Policy Satisfier(SPS) (HDFS-10285)](https://issues.apache.org/jira/browse/HDFS-10285)详细设计文档。

注意：当用户在目录上调用satisfyStoragePolicy() API时，SPS将扫描所有子目录并计算所有文件来满足策略。

HdfsAdmin API :` public void satisfyStoragePolicy(final Path path) throws IOException`

具体配置如下：

- **dfs.storage.policy.satisfier.mode** 用于启用NN外部的外部服务或禁用SPS。 支持以下类型：external, none。external表示启用了SPS，none表示禁用，默认none。
- **dfs.storage.policy.satisfier.recheck.timeout.millis** 重新检查datanode中已处理的块存储移动命令超时
- **dfs.storage.policy.satisfier.self.retry.timeout.millis** Datanode报告块移动结果的超时时间

### 3.2 Mover-一种新的数据迁移工具

添加了新的数据迁移工具来归档数据。 该工具类似于Balancer。 它会定期扫描HDFS中的文件，以检查块放置是否满足存储策略。 对于违反存储策略的块，它将副本复制到其他存储类型，以满足存储策略要求。 请注意，只要有可能，它总是尝试在同一节点内移动块副本。 如果这是不可能的(例如，当一个节点没有目标存储类型时)，它将通过网络将块副本复制到另一个节点。

命令如下：

```
hdfs mover [-p <files/dirs> | -f <local file name>]
```

参数说明：

| 名称                  | 描述                                                  |
| --------------------- | ----------------------------------------------------- |
| -p <files/dirs>       | 指定以空格分隔的要迁移的HDFS文件/目录列表             |
| -f < local file name> | 指定一个本地文件，其中包含要迁移的HDFS文件/目录的列表 |

注意：当同时省略-p和-f选项时，默认路径为根目录。

**管理员注意事项：**

StoragePolicySatisfier和Mover工具无法同时运行。 如果Mover实例已经触发并正在运行，则启动时将禁用SPS。 在这种情况下，管理员应确保Mover执行完成，然后再次启用外部SPS服务。 同样，如果已经启用SPS，则无法运行Mover。 如果管理员希望明确运行Mover工具，则应确保先禁用SPS，然后再运行Mover。 

## 4. 存储策略命令

### 4.1 列举存储策略

列举所有存储策略。

命令：

```
hdfs storagepolicies -listPolicies
```

### 4.2 设置存储策略

为文件或目录设置存储策略。

命令：

```
hdfs storagepolicies -setStoragePolicy -path <path> -policy <policy>
```

参数：

| 参数名称           | 描述                 |
| ------------------ | -------------------- |
| `-path <path>`     | 指向目录或文件的路径 |
| `-policy <policy>` | 存储策略名称         |

### 4.3 移除存储策略

取消对文件或目录的存储策略。 使用unset命令后，将应用最近祖先的存储策略，并且如果没有关于任何祖先的策略，则将应用默认存储策略。

```
hdfs storagepolicies -unsetStoragePolicy -path <path>
```

### 4.4 获取存储策略

获取文件或目录的存储策略。

```
hdfs storagepolicies -getStoragePolicy -path <path>
```

### 4.5 Satisfy Storage Policy

根据当前文件/目录存储策略调度移动块。

```
hdfs storagepolicies -satisfyStoragePolicy -path <path>
```

### 4.6 在NN外部启用外部服务或在不重新启动Namenode的情况下禁用SPS

如果管理员想在Namenode运行时切换SPS功能的模式。首先需要为配置文件(hdfs-site.xml)中的配置项`dfs.storage.policy.satisfier.mode`更新所需的值(external或none)。然后运行以下Namenode reconfig命令：

```
hdfs dfsadmin -reconfig namenode host:ipc_port start
```

### 4.7 外部开启SPS服务

如果管理员想启动外部sps。首先需要在配置文件(hdfs-site.xml)中使用external配置dfs.storage.policy.satisfier.mode属性，然后运行Namenode reconfig命令。 请确保配置文件中的网络拓扑配置与namenode相同，此群集将用于匹配目标节点。 之后，使用以下命令启动外部sps服务:

```
hdfs –daemon start sps
```

