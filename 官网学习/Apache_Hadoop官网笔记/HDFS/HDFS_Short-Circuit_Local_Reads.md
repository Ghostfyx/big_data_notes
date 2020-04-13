# HDFS Short-Circuit Local Reads

 ## 1. Short-Circuit Local Reads

### 1.2 背景介绍

在HDFS中，读取通常通过DataNode。 因此，当客户端要求DataNode读取文件时，DataNode将从磁盘上读取该文件，然后通过TCP套接字将数据发送给客户端。 所谓“Short-Circuit”读取绕过了DataNode，从而允许客户端直接读取文件。这仅在客户端与数据位于同一位置的情况下才可行。 Short-Circuit Read为许多应用提供了显着的性能提升。

### 1.2 开启

要配置短路本地读取，您将需要启用libhadoop.so。 有关启用此库的详细信息，请参见 [Native Libraries](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/NativeLibraries.html) 。

Short-circuit read使用UNIX域套接字。这是文件系统中的一条特殊路径，允许客户端和DataNode进行通信。 您将需要为此套接字设置一个路径。 DataNode需要能够创建此路径。 另一方面，除HDFS用户或root之外，任何其他用户都不可能创建该路径。 通常使用`/var/run`或`/var/lib`下的路径。

客户端和DataNode通过`/dev/shm`上的共享内存段交换信息。

Short-circuit read需要在namenode和客户端同时配置：

### 1.3 配置样例

```xml
<configuration>
  <property>
    <name>dfs.client.read.shortcircuit</name>
    <value>true</value>
  </property>
  <property>
    <name>dfs.domain.socket.path</name>
    <value>/var/lib/hadoop-hdfs/dn_socket</value>
  </property>
</configuration>
```

## 2. 旧版HDFS短路本地读取

客户端直接打开HDFS块文件的短路本地读取的传统实现仍可用于Linux以外的平台。将dfs.client.use.legacy.blockreader.local的值设置为true均可启用此功能。

需要将dfs.datanode.data.dir.perm的值设置为750，而不是默认值700，并chmod/chown dfs.datanode.data.dir下的目录树，以便客户端和DataNode可以读取。必须小心，因为这意味着客户端可以绕过HDFS权限读取所有阻止文件。

由于旧式短路本地读取不安全，因此只能在dfs.block.local-path-access.user值中列出的用户访问此功能：

```xml
<configuration>
  <property>
    <name>dfs.client.read.shortcircuit</name>
    <value>true</value>
  </property>
  <property>
    <name>dfs.client.use.legacy.blockreader.local</name>
    <value>true</value>
  </property>
  <property>
    <name>dfs.datanode.data.dir.perm</name>
    <value>750</value>
  </property>
  <property>
    <name>dfs.block.local-path-access.user</name>
    <value>foo,bar</value>
  </property>
</configuration>
```

