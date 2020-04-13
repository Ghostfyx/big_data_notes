# HDFS文件写入流程

下图详细描述了从新建文件，将数据写入文件，最终关闭该文件

![](../../img/3-4.jpg)

（1）客户端通过对`DistributedFileSystem`对象调用`create()`方法来创建文件；

（2）`DistributedFileSystem`对`NameNode`发起一次RPC调用，在文件系统命名空间创建一个新文件，此时文件还没有数据块。`NameNode`执行各种不同的检查以确保这个文件不存在以及客户端有新建该文件的权限。如果这些检查均通过，`NameNode`就会为创建新文件记录一条记录；否则，文件创建失败并向客户端抛出一个`IOException`异常。

（3）`DistributedFileSystem`向客户端返回一个`FSDataOutputStream`对象，`FSDataOutputStream`封装一个`DFSOutputStream`对象，`DFSOutputStream`对象负责处理`NameNode`与`DataNode`之间的通信。

（4）客户端向`DFSOutputStream`对象写入数据。

（5）客户端写入数据时，`DFSOutputStream`将它分成一个个数据包，并写入内部队列，称为数据队列(dataQueue)，`DataStreamer`处理数据队列，它的责任是挑选出适合存储数据副本的一组`DataNode`，并据此来要求`NameNode`分配新的数据块。这一组`DataNode`构成一个管线——我们假设副本数为3，所以管线中有3个节点。DataStreamer将数据包流流式传输到管线中第一个`DataNode`，该`DataNode`存储数据包并将它发送到管线中的第2个`DataNode`。同样，第2个`DataNode`存储该数据包并且发送给管线中的第3个（也是最后一个）`DataNode`；

（6）DFSOutputStream也维护着一个内部数据包队列来等待DataNode的收到确认回执，称为确认队列(ackQueue)，收到管道中所有DataNode确认信息后，该数据包才会从确认队列删除；

（7）客户端完成数据的写入后，对数据流调用`close()`方法；

（8）步骤7的操作将剩余的所有数据包写入`DataNode`管线，并在联系到`NameNode`告知其文件写入完成之前，等待确认。`NameNode`已经知道文件由哪些块组成（因为`DataStreamer`请求分配数据块），所以它在返回成功前只需要等待数据块进行最小量的复制；

（9）数据写入流程结。

如果任何`DataNode`在数据写入期间发生故障，则执行以下操作：

（1）关闭管线，把确认队列中的所有数据包都添加回数据队列的最前端，以确保故障节点下游的`DataNode`不会漏掉任何一个数据包；

（2）为存储在另一个正常`DataNode`的当前数据块指定一个新的标识，并将该标识传给`NameNode`，以便故障`DataNode`在恢复后可以删除存储的部分数据块；

（3）从管线删除故障`DataNode`，基于两个正常`DataNode`构建新管线，余下的数据块写入管线中正常的`DataNode`。`NameNode`注意到块副本量不足时，会在另一个节点上创建一个新的副本。后续的数据块继续正常接受处理。

在一个块被写入期间可能会有多个`DataNode`同时发生故障，但非常少见。主要写入了`dfs.namenode.replication.min`的副本数（默认为1），写操作就会成功，并且这个块可以在集群中异步复制，直到达到其目标副本数（dfs.replication的默认值为3）。

