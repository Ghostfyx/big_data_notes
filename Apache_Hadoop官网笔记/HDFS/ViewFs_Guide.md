# ViewFs Guide

## 1. 简介

视图文件系统(View File System，ViewFs)提供了一种管理多个Hadoop文件系统命名空间(或名称空间卷)的方法。在联邦HDFS中，对于具有多个Namenode(因此具有多个命名空间卷)的集群而言，特别有用。ViewFs类似于某些Unix / Linux系统中的客户端挂载表。 ViewF可以用于创建个性化的名称空间视图，也可以用于每个集群的通用视图。

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

或者，可以通过linkMergeSlash将装载表的根与另一个文件系统的根合并。在以下安装表配置中，在`hdfs://nn1-clustery.example.com:8020`上将Cluster Y的根目录与根文件系统合并。

```xml
<configuration>
  <property>
    <name>fs.viewfs.mounttable.ClusterY.linkMergeSlash</name>
    <value>hdfs://nn1-clustery.example.com:8020/</value>
  </property>
</configuration>
```

### 3.3 路径名使用方式

由此在集群X上，将core-site.xml设置为使默认fs使用该群集的安装表，因此典型路径名是：

1. `/foo/bar`

	此路径与`viewfs://clusterX/foo/bar` 等价，如果在非联邦HDFS的旧版本中使用这样的路径名，那么向联合HDFS过度是透明的。

2. `viewfs://clusterX/foo/bar`

	尽管这是一个有效的路径名，但最好使用`/foo/bar`，因为它允许在需要时将应用及其数据透明地移动到另一个集群。

3. `viewfs://clusterY/foo/bar`

	它是用于引用另一个群集(例如群集Y)上路径名的URI。特别是，用于将文件从群集Y复制到群集Z的命令如下所示：

	```
	distcp viewfs://clusterY/pathSrc viewfs://clusterZ/pathDest
	```

4. `viewfs://clusterX-webhdfs/foo/bar`

	通过WebHDFS文件系统访问文件的URI

5.  http://namenodeClusterX:http_port/webhdfs/v1/foo/bar` and `http://proxyClusterX:http_port/foo/bar

	用于通过WebHDFS REST API和HDFS代理访问文件。 

### 3.4 路径名称使用最佳实践

当一个在群集中时，建议使用上面类型(1)的路径名，而不是像(2))这样的标准URI。此外，应用程序不应使用挂载点的知识和hdfs://namenodeContainingUserDirs:port/joe/foo/bar 之类的路径来引用特定namenode中的文件。应该使用`/user/joe/foo/bar`代替。

### 3.5 跨命名空间重命名路径

回想一下，不能在旧版本的名称节点或群集之间重命名文件或目录。 在新版本中也是如此，但有一个额外的变化。 例如，在旧版本中，可以执行以下命令。

```
rename /user/joe/myStuff /data/foo/bar
```

在新版本中，如果/user和/data实际上存储在集群中的不同namenode上，则命令不起作用。

## 4. 具有Nfs挂载点的多文件系统I/O

HDFS和其他分布式文件系统通过某种冗余(例如：块复制或更复杂的分布式编码)提供数据弹性。但是，当前企业级Hadoop平台可能由多个Hadoop群集，企业文件管理器组成，它们托管在内部和外部。Nfly挂载点使单个逻辑文件可以被多个文件系统同步复制。 它是为较小的文件而设计的，最大容量为GB。 通常，这是单个核心/单个网络链接性能的函数，因为它位于使用ViewF(例如FsShell或MapReduce任务)的单个客户端JVM中。

### 4.1 基本配置

考虑以下示例，以了解Nfly的基本配置。假设要在URI表示的三个文件系统上保留广告：uri1，uri2和uri3。

```xml
<property>
    <name>fs.viewfs.mounttable.global.linkNfly../ads</name>
    <value>uri1,uri2,uri3</value>
</property>
```

注意属性名称中连续2个**.**。它们的出现是由于对安装点进行高级调整的空设置，将在后续部分中显示。 该属性值是逗号分隔的URI列表。

URI可能指向不同区域中的不同群集：`hdfs://datacenter-east/ads, s3a://models-us-west/ads, hdfs://datacenter-west/ads `或者最简单的情况是指向相同的文件系统，例如：`file:/tmp/ads1, file:/tmp/ads2, file:/tmp/ads3`。

如果基础系统可用，则在全局路径viewfs://global/ads下执行的所有修改都会传播到所有目标URI。

例如，如果通过hadoop shell创建文件：

```sh
hadoop fs -touchz viewfs://global/ads/z1
```

将在后一种配置中通过本地文件系统找到它：

```
ls -al /tmp/ads*/z1
-rw-r--r--  1 user  wheel  0 Mar 11 12:17 /tmp/ads1/z1
-rw-r--r--  1 user  wheel  0 Mar 11 12:17 /tmp/ads2/z1
-rw-r--r--  1 user  wheel  0 Mar 11 12:17 /tmp/ads3/z1
```

第一个文件系统处理从全局路径读取的操作，不会导致异常。 文件系统的访问顺序取决于当前是否可用，以及是否存在拓扑顺序。

### 4.2 进阶配置

挂载点linkNfly可以使用作为键=值对的逗号分隔列表传递的参数进行进一步配置。当前支持以下参数。

如果以下nfly写入失败，则minReplication = int确定必须处理写入修改而没有异常的最小目标数。如果minReplication高于目标URI的数量，则配置错误。默认值为2。

如果minReplication低于目标URI的数量，我们可能会有一些目标URI没有最新写入。可以通过采用以下设置控制的更昂贵的读取操作来补偿它。

readMostRecent = boolean如果设置为true，则Nfly客户端将检查所有目标URI下的路径，而不是仅根据拓扑顺序检查第一个目标URI。在当前所有可用的文件中，处理时间最近的文件将被处理。

### 4.3 网络拓扑结构

Nfly试图满足对“最接近”目标URI的读取。为此，Nfly将“机架感知”概念扩展到目标URI的权限。

Nfly应用NetworkTopology来解析URI的权限。 最常见的是，在异构设置中使用基于脚本的映射。 可以使用一个脚本，提供以下拓扑映射：

| URI                          | Topology                |
| :--------------------------- | :---------------------- |
| `hdfs://datacenter-east/ads` | /us-east/onpremise-hdfs |
| `s3a://models-us-west/ads`   | /us-west/aws            |
| `hdfs://datacenter-west/ads` | /us-west/onpremise-hdfs |

如果目标URI不具有 file:/中的授权部分，则Nfly将注入客户端的本地节点名称。

### 4.4 Nfly配置示例

```xml
<property>
    <name>fs.viewfs.mounttable.global.linkNfly.minReplication=3,readMostRecent=true,repairOnRead=false./ads</name>
    <value>hdfs://datacenter-east/ads,hdfs://datacenter-west/ads,s3a://models-us-west/ads,file:/tmp/ads</value>
  </property>
```

### 4.5 Nfly文件如何开始工作

```java
FileSystem fs = FileSystem.get("viewfs://global/", ...);
FSDataOutputStream out = fs.create("viewfs://global/ads/f1");
out.write(...);
out.close();
```

上面的代码将执行以下逻辑：

1. 在每个目标URI下创建一个不可见文件`_nfly_tmp_f1`，即 `hdfs://datacenter-east/ads/_nfly_tmp_f1, hdfs://datacenter-west/ads/_nfly_tmp_f1`, 这是通过在基础文件系统上调用create并返回FSDataOutputStream来完成，包装所有四个输出流的对象。
2. 随后的写出可以转发到每个包装的流
3. 在out.close上，所有流都关闭，文件从_nfly_tmp_f1重命名为f1。 截至此步骤开始，所有文件都将收到与客户端系统时间相对应的相同修改时间。
4. 如果至少minReplication目标已经通过步骤1-3且没有失败，则Nfly认为该事务是逻辑提交的； 否则，它会尽力尝试清除临时文件。

请注意，由于4是尽力而为的步骤，并且客户端JVM可能崩溃并且永远无法恢复工作，因此，最好提供某种cron作业来清除此类_nfly_tmp文件。

## 5. FAQ

1. 当从非联邦HDFS版本迁移到联邦HDFS版本时，将不得不跟踪不同卷的namenode。该怎么做？

	不会。请参阅上面的示例——正在使用相对名称并利用默认文件系统，或者将路径从hdfs://namenodeCLusterX/foo/bar更改为viewfs://clusterX/foo/bar。

2. 操作将某些文件从群集中的一个namenode移动到另一个namenode会发生什么？

	操作可能会将文件从一个名称节点移到另一个名称节点，以处理存储容量问题。将使用某种方式避免正在运行的应用中断，例如：

	- 示例1：/user和/data在一个namenode上，后来它们需要在单独的namenode上以处理容量问题。实际上，操作将为/user和/data创建单独的挂载点。更改之前，/user和/data的挂载将指向相同的namenode，例如namenodeContainingUserAndData。操作将更新挂载表，以便将挂载点分别更改为namenodeContaingUser和namenodeContainingData。
	- 示例2：所有项目都挂载在一个namenode上，但后来它们需要两个或多个namenode。 ViewFs允许像/project/foo和/project/bar这样的挂载。这允许将挂载表更新为指向相应的namenode。

3. 挂载表是位于每个core-site.xml中还是位于其自己的单独文件中？

	计划是将挂载表保存在单独的文件中，并使core-site.xml包含其中。尽管可以将这些文件本地保存在每台机器上，但最好使用HTTP从中央位置访问它。

4. 配置是否应该仅对一个集群或所有集群都具有挂载表定义？

	该配置应具有所有集群的安装定义，因为一个集群需要访问其他集群中的数据，例如distcp。

5. 考虑到Operations可能会随着时间更改挂载表，何时实际读取挂载表？

	将作业提交到集群后，将读取挂载表。提交作业时会扩展core-site.xml中的XInclude。这意味着如果更改了挂载表，则需要重新提交作业。由于这个原因，要实现合并挂载，这将大大减少更改挂载表的需求。此外，希望通过将来在作业开始时初始化的另一种机制读取挂载表。

6. JobTracker（或Yarn的资源管理器）本身会使用ViewF吗？

	不需要。 NodeManager也不需要

7. ViewFs是否仅允许在最高级别进行安装？

	没有;更一般。例如，可以挂载/user/joe和/user jane。在这种情况下，将在挂载表中为/user创建一个内部只读目录。 /user上的所有操作均有效，但/user是只读的。

8. 应用程序可跨群集工作，并且需要永久存储文件路径。它应该存储哪些路径？

	应该存储`viewfs://cluster/path`类型的路径名，与运行应用程序时使用的路径名相同。只要操作以透明的方式进行，这就能避免数据在集群内名称节点内的移动。如果数据从一个群集移动到另一个群集，它不会与外界隔离。无论如何，旧版本(联合HDFS之前)都无法保护形成跨集群的此类数据移动。

9. 委托令牌呢？

	要向其提交作业的集群的委托令牌(包括该集群挂载表的所有已挂载卷)，对于应用的map-reduce作业的输入和输出路径(包括通过挂载表为指定的输入和输出路径挂载的所有卷)都将自动进行处理。此外，在特殊情况下，还有一种方法可以将其他委托令牌添加到基本群集配置中。

## 6. 附录：挂载表配置示例

通常，用户不必定义挂载表或core-site.xml即可使用该挂载表。 这是通过操作完成的，并且在正确的网关机器上设置了正确的配置，就像对core-site.xml所做的那样。

可以在core-site.xml中配置挂载表，但是最好在core-site.xml中使用间接引用来引用单独的配置文件，例如mountTable.xml。 将以下配置元素添加到core-site.xml中，以引用mountTable.xml：

```xml
<configuration xmlns:xi="http://www.w3.org/2001/XInclude"> 
  <xi:include href="mountTable.xml" />
</configuration> 
```

在文件mountTable.xml中，为虚拟集群“ClusterX”定义了一个挂载表，该集群是一个由三个namenodes管理的三个命名空间卷的联合：

```
nn1-clusterx.example.com:8020,
nn2-clusterx.example.com:8020,
nn3-clusterx.example.com:8020,
```

/home和/tmp在namenode(nn1-clusterx.example.com:8020)管理的命名空间中，/foo和/bar托管在联合集群的其他namenodes上 。主目录基本路径设置为/home，以便每个用户都可以使用FileSystem/FileContext中定义的`getHomeDirectory()`方法访问其主目录。

```xml
<configuration>
  <property>
    <name>fs.viewfs.mounttable.ClusterX.homedir</name>
    <value>/home</value>
  </property>
  <property>
    <name>fs.viewfs.mounttable.ClusterX.link./home</name>
    <value>hdfs://nn1-clusterx.example.com:8020/home</value>
  </property>
  <property>
    <name>fs.viewfs.mounttable.ClusterX.link./tmp</name>
    <value>hdfs://nn1-clusterx.example.com:8020/tmp</value>
  </property>
  <property>
    <name>fs.viewfs.mounttable.ClusterX.link./projects/foo</name>
    <value>hdfs://nn2-clusterx.example.com:8020/projects/foo</value>
  </property>
  <property>
    <name>fs.viewfs.mounttable.ClusterX.link./projects/bar</name>
    <value>hdfs://nn3-clusterx.example.com:8020/projects/bar</value>
  </property>
</configuration>
```

