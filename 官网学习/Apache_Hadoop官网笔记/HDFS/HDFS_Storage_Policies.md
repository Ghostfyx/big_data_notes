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



