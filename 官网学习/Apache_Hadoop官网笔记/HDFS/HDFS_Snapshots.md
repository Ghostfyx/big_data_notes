# HDFS Snapshots

 ## 1.  概述

HDFS Snapshots是文件系统的只读时间点副本。 可以在文件系统或整个文件系统的子树上形成快照。 快照的一些常见用例是数据备份，防止用户错误和灾难恢复。

HDFS的快照功能是非常高效的：

- 快照创建是瞬时的：不包括inode查找时间，成本为O(1)
- 仅当相对于快照进行修改时才使用附加内存：内存使用量为O(M)，其中M是已修改文件/目录的数量
- 不会复制datanode中的块：快照文件记录了块列表和文件大小，没有数据复制
- 快照不会对常规HDFS操作产生不利影响：修改记录以相反的时间顺序进行，因此可以直接访问当前数据。 通过从当前数据中减去修改来计算快照数据

### 1.1 快照目录

快照可以在任何目录上存储，只需将目录设置为快照表。 快照表目录能够容纳65536个同时快照。 快照表目录的数量没有限制。 管理员可以将任何目录设置为快照表。 如果快照表目录中有快照，则在删除所有快照之前，不能删除或重命名该目录。

### 1.2 快照路径

对于快照表目录，路径组件`.snapshot`用于访问其快照。假设`/foo`是快照表目录，`/foo/bar`是`/foo`中的文件目录。并且`/foo`具有快照$S_0$。路径`/foo/.snapshot/s0/bar`引用`/foo/bar`的快照复本，常用的API和CLI可以使用“ .snapshot”路径。 以下是一些示例：

- 列出snapshottable目录下的所有快照：

	```
	hdfs dfs -ls /foo/.snapshot
	```

- 列出快照$S_0$下的文件：

	```
	hdfs dfs -ls /foo/.snapshot/s0
	```

- 从快照$S_0$复制文件：

	```
	hdfs dfs -cp -ptopax /foo/.snapshot/s0/bar /tmp
	```

	请注意，使用preserve选项保留时间戳，所有权，权限，ACL和XAttrs。

## 2. 升级到带有快照的HDFS版本

HDFS快照功能引入了用于与快照进行交互的新的保留路径名：`.snapshot`。HDFS快照功能引入了用于与快照进行交互的新的保留路径名：`.snapshot`。有关更多信息，请参见 [the HDFS user guide](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HdfsUserGuide.html#Upgrade_and_Rollback)中的升级部分。

## 3. 快照操作

### 3.1 管理员操作

本节中描述的操作需要超级用户权限。

#### 3.1.1 开放快照功能

允许创建目录快照。 如果操作成功完成，目录将成为快照表。

- 命令

	```
	hdfs dfsadmin -allowSnapshot <path>
	```

- 参数

	| 参数名称 | 参数描述   |
	| -------- | ---------- |
	| path     | 快照表目录 |

另请参见HdfsAdmin中相应的Java API `void allowSnapshot(Path path)`。

### 3.1.2 不允许快照功能

不允许创建目录的快照。 必须先删除目录的所有快照，然后再禁止快照。

- 命令

	```
	hdfs dfsadmin -disallowSnapshot <path>
	```

- 参数

	| 参数名称 | 参数描述   |
	| -------- | ---------- |
	| path     | 快照表目录 |

另请参见HdfsAdmin中相应的Java API `void disallowSnapshot(Path path)`。

### 3.2 用户操作

本节介绍用户操作。 请注意，HDFS超级用户可以执行所有操作，而无需满足单个操作中的权限要求。

#### 3.2.1 创建快照

创建快照表目录的快照。 此操作需要快照表目录的所有者权限。

- 命令

	```
	hdfs dfs -createSnapshot <path> [<snapshotName>]
	```

- 参数

	| 参数         | 描述                                                         |
	| ------------ | ------------------------------------------------------------ |
	| path         | 快照表目录                                                   |
	| snapshotName | 快照名称，这是一个可选参数。 省略时，将使用带有以下格式的时间戳生成默认名称 `"'s'yyyyMMdd-HHmmss.SSS"`, e.g. `"s20130412-151029.033"`. |

另请参见FileSystem中相应的Java API `Path createSnapshot(Path path) `与`Path createSnapshot(Path path, String snapshotName)`，方法中返回快照路径。

#### 3.2.2 删除快照

从快照表目录中删除快照。 此操作需要快照表目录的所有者权限。

- 命令

	```
	hdfs dfs -deleteSnapshot <path> <snapshotName>
	```

- 参数

	| 参数         | 描述       |
	| ------------ | ---------- |
	| path         | 快照表目录 |
	| snapshotName | 快照名称   |

另请参见[FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html)中相应的Java API `void deleteSnapshot(Path path, String snapshotName)`。

#### 3.2.3 重命名快照

重命名快照。 此操作需要快照表目录的所有者权限。

- 命令

	```
	hdfs dfs -renameSnapshot <path> <oldName> <newName>
	```

- 参数

	| 参数    | 描述       |
	| ------- | ---------- |
	| path    | 快照表目录 |
	| oldName | 旧快照名称 |
	| newName | 新快照名称 |

另请参见[FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html)中相应的Java API `void renameSnapshot(Path path, String oldName, String newName)`。

#### 3.2.4 获取Snapshottable目录列表

获取当前用户有权限的快照的所有快照表目录。

```
hdfs lsSnapshottableDir
```

另请参见`DistributedFileSystem`中相应的Java API `SnapshottableDirectoryStatus [] `和`getSnapshottableDirectoryListing()`。

#### 3.2.5 获取快照差异报表

获取两个快照之间的差异。 此操作需要两个快照中所有文件/目录的读取访问权限。

- 命令

	```
	hdfs snapshotDiff <path> <fromSnapshot> <toSnapshot>
	```

- 参数

	| 参数         | 描述         |
	| ------------ | ------------ |
	| path         | 快照表目录   |
	| fromSnapshot | 起始快照名称 |
	| toSnapshot   | 终止快照名称 |

	请注意，snapshotDiff可用于获取两个快照之间或快照与目录的当前状态之间的差异报告。 用户可以使用“.” 代表当前状态。

- 结果

	| 符号 | 代表含义                              |
	| ---- | ------------------------------------- |
	| +    | The file/directory has been created.  |
	| -    | The file/directory has been deleted.  |
	| M    | The file/directory has been modified. |
	| R    | The file/directory has been renamed.  |

RENAME条目指示文件/目录已被重命名，但仍在同一快照表目录下。如果将文件/目录重命名为快照目录的外部，则报告为已删除。 从快照目录外部重命名的文件/目录被报告为新创建的。

快照差异报告不保证相同的操作顺序。 例如，如果将目录“/foo”重命名为“/foo2”，然后将新数据追加到文件“/foo2/bar”，则差异报告将为：

```
R. /foo -> /foo2
M. /foo/bar
```

即，报告中的重命名目录下文件/目录的更改是使用重命名之前的原始路径(在上面的示例中为“/foo/bar”)。

另请参见DistributedFileSystem中相应的Java API `SnapshotDiffReport getSnapshotDiffReport(Path path, String fromSnapshot, String toSnapshot)`

