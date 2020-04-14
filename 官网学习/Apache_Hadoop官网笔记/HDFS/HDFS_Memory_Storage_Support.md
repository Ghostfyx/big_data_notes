# Memory Storage Support in HDFS

 ## 1. 简介

HDFS支持写入由datanode管理的堆外内存。数据节点将异步将内存中的数据刷新到磁盘，从性能敏感的IO路径中剔除了昂贵的磁盘IO和校验和计算，将这种写入称为**异构持久化写入(Lazy Persist Write)**。HDFS为异构持久化写入提供了最大化的保障，如果在将副本保留到磁盘之前重新启动节点，则可能会丢失很少的数据。 应用程序可以选择使用持久写入来权衡一些持久性保证，以减少延迟。

此功能从Apache Hadoop 2.6.0开始可用。

![](../img/LazyPersistWrites.jpg)

Lazy Persisr Write受益目标是低延迟写入相对少量数据(从几GB到数十GB，具体取决于可用内存)的应用程序。内存存储用于在集群内运行并与位于HDFS DataNode的应用程序。因为网络复制的延迟开销抵消了写入内存的好处。即：应用运行所在的datanode可使用内存存储数据。

如果内存不足或未配置，则使用延迟写入的应用程序将继续工作，方法是退回到DISK存储。

## 2. 管理员配置

### 2.1 限制用于内存中副本的RAM

确定用于存储内存副本的内存量。在hdfs-site.xml中设置dfs.datanode.max.locked.memory。 这与集中式缓存管理功能所使用的设置相同。数据节点将确保异步持久写入和集中式缓存管理使用的组合内存不超过dfs.datanode.max.locked.memory中配置的数量。

例如：为内存中副本保留32GB空间：

```xml
<property>
  <name>dfs.datanode.max.locked.memory</name>
  <value>34359738368</value>
</property>
```

启动时，datanode未分配此内存。

在类似Unix的系统上，还需要增加数据节点用户的内存大小`ulimit(ulimit -l)` 。设置此值时，还需要在内存中留出其他空间，例如数据节点和应用程序JVM堆以及操作系统页面缓存。 如果在与数据节点相同的节点上运行YARN NodeManager进程，则还将需要考虑YARN容器的内存。

### 2.2 在数据节点上设置RAM磁盘

在每个数据节点上初始化一个RAM磁盘。 RAM磁盘的选择可在数据节点进程重启时实现更好的数据持久性。 以下设置适用于大多数Linux发行版。 当前不支持在其他平台上使用RAM磁盘。

#### 2.2.1 选择tmpfs还是ramfs

Linux支持使用两种RAM磁盘-tmpfs和ramfs。 Linux内核限制了tmpfs的大小，而ramfs逐渐增长以占用所有可用的系统内存。tmpfs有一个缺点，因为它的内容可以在内存压力下交换到磁盘上。 但是，许多对性能敏感的部署在禁用交换的情况下运行，因此这在实际中不会成为问题。

HDFS当前支持使用tmpfs分区。 正在支持添加ramfs。

#### 2.2.2 挂载RAM磁盘

使用Unix mount命令挂载RAM磁盘分区。 例如：在/mnt/dn-tmpfs/下挂载32Gtmpfs分区：

```
   sudo mount -t tmpfs -o size=32g tmpfs /mnt/dn-tmpfs/
```

建议在/etc/fstab下创建一条挂载条目，以便在节点重新启动时自动重新挂载RAM磁盘。 另一个选择是使用/dev/shm下的子目录，该目录是大多数Linux发行版默认可用的tmpfs挂载。 确保挂载大小大于或等于dfs.datanode.max.locked.memory设置，否则在/etc/fstab中覆盖它。 建议不要在每个数据节点上使用多个tmpfs分区进行异步持久写入。

### 2.2.3 用RAM_DISK存储类型标记tmpfs卷

通过hdfs-site.xml中的dfs.datanode.data.dir配置设置，用RAM_DISK存储类型标记tmpfs目录。 例如。 在具有三个硬盘卷/grid/0, /grid/1和/grid/2以及tmpfs挂载/mnt/dn-tmpfs的datanode上，dfs.datanode.data.dir必须设置如下：

```xml
<property>
  <name>dfs.datanode.data.dir</name>
  <value>/grid/0,/grid/1,/grid/2,[RAM_DISK]/mnt/dn-tmpfs</value>
</property>
```

此步骤至关重要。 如果没有RAM_DISK标签，HDFS将把tmpfs卷视为非易失性存储，并且数据将不会保存到持久化存储中。 将在节点重启时丢失数据。

#### 2.2.4 确保存储策略开启

确保打开启用存储策略的全局设置。 默认情况下，此设置为启用。

## 3. 应用示例

### 3.1 使用LAZY_PERSIST存储策略

应用程序表明HDFS可以将具有LAZY_PERSIST存储策略的文件用于异步延迟写入。 设置策略不需要管理权限，可以通过以下三种方法之一进行设置。

### 3.1.1命令行设置

```
    hdfs storagepolicies -setStoragePolicy -path <path> -policy LAZY_PERSIST
```

#### 3.1.2 调用目录的setStoragePolicy方法

从Apache Hadoop 2.8.0开始，应用程序可以通过FileSystem.setStoragePolicy以编程方式设置存储策略。 例如：

```java
    fs.setStoragePolicy(path, "LAZY_PERSIST");
```

#### 3.1.3 创建新文件使用CreateFlag

使用FileSystem＃create API创建新文件时，应用程序可以使用参数CreateFlag.LAZY_PERSIST。 例如：

```java
 FSDataOutputStream fos =
        fs.create(
            path,
            FsPermission.getFileDefault(),
            EnumSet.of(CreateFlag.CREATE, CreateFlag.LAZY_PERSIST),
            bufferLength,
            replicationFactor,
            blockSize,
            null);
```

