# ViewFs Guide

## 1. 简介

View File System(ViewFs)提供了一种管理多个Hadoop文件系统命名空间(或名称空间卷)的方法。在联邦HDFS中，对于具有多个Namenode(因此具有多个命名空间卷)的集群而言，特别有用。ViewFs类似于某些Unix / Linux系统中的客户端挂载表。 ViewF可以用于创建个性化的名称空间视图，也可以用于每个集群的通用视图。

## 2. 旧版本(联邦HDFS之前)

### 2.1 单Namenode集群

在联合HDFS之前的版本中，集群只有单个namenode，单个nameNode为集群提供了单个文件系统命名空间。假设有多个集群。 每个群集的文件系统命名空间是完全独立且不相交的， 此外，物理存储不跨集群共享(即数据节点不跨集群共享)。

每个集群的core-site.xml具有一个配置属性——该属性将默认文件系统设置为该集群的namenode：

```xml
<property>
  <name>fs.default.name</name>
  <value>hdfs://namenodeOfClusterX:port</value>
</property>
```

这种配置属性允许使用斜杠相对名称来解析相对于群集namenode的路径。 例如，使用上述配置，路径`/foo/bar`指向`hdfs://namenodeOfClusterX：port/foo/bar`。在群集上的每个网关以及该群集的关键服务(例如JobTracker和Oozie)上都设置了此配置属可以使用上述 特性。

### 2.2 路径名使用方式

在如上设置core-site.xml的集群X上，典型路径名是：

1. /foo/bar

	与以前一样，等效于`hdfs://namenodeOfClusterX:port/foo/bar`

2. `hdfs://namenodeOfClusterX:port/foo/bar`

	尽管这是一个有效的路径名，但最好使用`/foo/bar`，因为它允许在需要时将应用及其数据透明地移动到另一个集群(只需要修改core-site.xml配置文件)。

3. `hdfs://namenodeOfClusterY:port/foo/bar`

	用于引用另一个集群(例如：集群Y)上的路径名。下面命令将集群Y上的文件复制到集群X上：

	```shell
	distcp hdfs://namenodeClusterY:port/pathSrc hdfs://namenodeClusterZ:port/pathDest
	```

4. `webhdfs://namenodeClusterX:http_port/foo/bar`

	访问webHDFS文件系统中文件的URI，**注意：**webHDFS系统使用的是nameNode的HTTP端口而不是RPC端口。

5. http://namenodeClusterX:httpport/webhdfs/v1/foo/bar` and `http://proxyClusterX:httpport/foo/bar

	上述分别是HTTP URL，通过WenHDFS REST API和HDFS代理访问文件。

### 2.3 文件路径最佳实践

当一个在群集中时，建议使用上面类型(1)的路径名，而不是像(2)这样的标准URI。 完全限定的URI与地址类似，并且不允许应用随其数据一起移动。

## 3. 新版本——联邦HDFS与ViewFs

### 3.1 如何查找集群

假设有多个集群，每个集群有一个或多个namenode，每个namenode有其自己的命名空间。一个namenode仅属于一个集群。同一个集群中的namenode共享集群中的物理存储空间。群集之间的命名空间与以前一样是独立的。

操作会根据存储需求决定集群中每个namenode的存储内容。例如，他们可以将所有用户数据(/user/<username>)放在一个namenode中，所有feed数据(/data)放在另一个namenode中，所有项目(/projects)放在另一个namenode中，等等。

### 3.2 使用ViewFs的每个群集全局命名空间

为了提供对旧版本的透明性，ViewFs文件系统(客户端安装)用于为每个集群创建一个独立的集群namespace视图，该视图与旧版本中的namespace类似。客户端侧安装表类似于unix安装表，它们使用旧的命名空间约定安装新的命名空间卷。下图显示了一个装载表，该装载表装载了四个名称空间卷——`/user`，`/data`，`/projects`和`/tmp`：

![](../img/viewfs_TypicalMountTable.jpg)

ViewFs实现了Hadoop文件系统接口，就像HDFS和本地文件系统。从仅允许链接到其他文件系统的意义上来说，它是一个琐碎的文件系统。由于ViewFs实现Hadoop文件系统接口，因此透明地运行Hadoop工具。 例如，所有的shell命令都可以与HDFS和本地文件系统一起使用ViewFs。

在每个集群的配置中，如下所示，将默认文件系统设置为该集群的装入表(与[Single Namenode Clusters](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/ViewFs.html#Single_Namenode_Clusters)集群中的配置进行比较)。

```xml
<property>
  <name>fs.defaultFS</name>
  <value>viewfs://clusterX</value>
</property>
```

URI中的`viewfs://scheme`后面的权限是装入表名称。建议使用集群名称来命名集群的安装表，然后，Hadoop系统将在Hadoop配置文件中查找名称为“ clusterX”的装载表。操作将所有网关和服务机安排为包含所有群集的装入表，以便如上所述，对于每个群集，默认文件系统均设置为该群集的ViewFs装入表。

装载表的挂载点在标准Hadoop配置文件中指定，viewfs的所有安装表配置条目均以fs.viewfs.mounttable为前缀。使用link标签指定连接其他文件系统挂载点。建议使用与链接文件系统目标位置中相同的挂载点名称。对于安装表中未配置的所有命名空间，可以通过linkFallback将它们回退到默认文件系统。

在下面装载表配置中，/data命名空间链接到文件系统`hdfs://nn1-clusterx.example.com:8020/data`，`/project` is linked to the filesystem `hdfs://nn2-clusterx.example.com:8020/project`。所有未在安装表中配置的名称空间（例如/logs）都链接到文件系统`hdfs://nn5-clusterx.example.com:8020/home`。

```xml
<configuration>
  <property>
    <name>fs.viewfs.mounttable.ClusterX.link./data</name>
    <value>hdfs://nn1-clusterx.example.com:8020/data</value>
  </property>
  <property>
    <name>fs.viewfs.mounttable.ClusterX.link./project</name>
    <value>hdfs://nn2-clusterx.example.com:8020/project</value>
  </property>
  <property>
    <name>fs.viewfs.mounttable.ClusterX.link./user</name>
    <value>hdfs://nn3-clusterx.example.com:8020/user</value>
  </property>
  <property>
    <name>fs.viewfs.mounttable.ClusterX.link./tmp</name>
    <value>hdfs://nn4-clusterx.example.com:8020/tmp</value>
  </property>
  <property>
    <name>fs.viewfs.mounttable.ClusterX.linkFallback</name>
    <value>hdfs://nn5-clusterx.example.com:8020/home</value>
  </property>
</configuration>
```

或者，可以通过linkMergeSlash将装载表的根与另一个文件系统的根合并。在以下安装表配置中，在`hdfs://nn1-clustery.example.com:8020`上将ClusterY的根目录与根文件系统合并。

```xml
<configuration>
  <property>
    <name>fs.viewfs.mounttable.ClusterY.linkMergeSlash</name>
    <value>hdfs://nn1-clustery.example.com:8020/</value>
  </property>
</configuration>
```

