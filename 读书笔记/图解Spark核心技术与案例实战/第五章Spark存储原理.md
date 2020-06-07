# 第五章 Spark存储原理

## 5.1 存储分析

本节将对Spark的存储进行全面的分析，Spark存储介质包括存储和磁盘等等。通过本节的分歧，我们可以了解到Spark在不同运行场景下读写数据过程。

### 5.1.1 整体架构

Spark的存储信息采取了主从模式，即Master/Slave模式，整个存储模块使用了第三节介绍的RPC的通信方式。其中，Master负责整个应用程序运行期间的数据块元数据的管理和维护，而Slave一方面负责将本地数据块的信息上报给Master，另一方面接收从Master闯过来的执行命令，如获取数据块状态、删除RDD/数据块等命令。在每个Slave存在数据传输通道，根据需要在Slave之间进行远程数据的读取和写入。Spark的存储整体架构如图5-1所示。

<img src="img/5-1.jpg" style="zoom:50%;" />

根据Spark存储整体架构图，下面将根据数据生命周期过程进行消息通信：

（1）在应用程序启动时，SparkContext会创建Driver端的SparkEnv，在该SparkEnv中实例化BlockManager和BlockManagerMaster，在BlockManagerMaster内部创建消息通信的终端点BlockManagerMasterEndpoint。

在Executor启动时也会创建其SparkEnv，在该SparkEnv中实例化BlockManager和负责网络数据传输服务的BlockTransferService，在BlockManager初始化过程中，一方面会加入BlockManagerSlaveEndpoint终端点，并把该终端点的应用注册到Driver中，这样Driver和Executor互相持有通信端点的引用，可以在应用执行过程中进行消息通信。实例化BlockTransferService过程中，使用Netty的数据传输服务方式(在Spark 2.0版本之前，提供Netty和NIO进行选择)。由于该数据传输服务隐藏了集群间不同节点间的消息传出操作，可类似于本地数据操作方式进行数据读写，大大简化了网络数据传输的复杂过程。在SparkEnv类中创建BlockTransferService、BlockManager和BlockManagerMaster代码如下：

```scala
/**
      * 创建BlockManagerMaster，如果是Driver端在BlockManagerMaster内部，则创建
      * BlockManagerMasterEndpoint，如果是Executor，则创建BlockManagerMasterEndpoint的引用
      */
val blockManagerMaster = new BlockManagerMaster(
  registerOrLookupEndpoint(
    BlockManagerMaster.DRIVER_ENDPOINT_NAME,
    new BlockManagerMasterEndpoint(
      rpcEnv,
      isLocal,
      conf,
      listenerBus,
      if (conf.get(config.SHUFFLE_SERVICE_FETCH_RDD_ENABLED)) {
        externalShuffleClient
      } else {
        None
      }, blockManagerInfo)),
  registerOrLookupEndpoint(
    BlockManagerMaster.DRIVER_HEARTBEAT_ENDPOINT_NAME,
    new BlockManagerMasterHeartbeatEndpoint(rpcEnv, isLocal, blockManagerInfo)),
  conf,
  isDriver)

// 创建远程数据传输服务，使用Netty方式
val blockTransferService =
new NettyBlockTransferService(conf, securityManager, bindAddress, advertiseAddress,
                              blockManagerPort, numUsableCores, blockManagerMaster.driverEndpoint)

// NB: blockManager is not valid until initialize() is called later.
// 创建BlockManager，如果是Driver端包含BlockManagerMaster，如果是Executor包含的是blockManagerMaster的引用
// 另外BlockManagerMaster包含了远程数据传输服务，当BlockManager调用initialize()方法初始化时真正生效
val blockManager = new BlockManager(
  executorId,
  rpcEnv,
  blockManagerMaster,
  serializerManager,
  conf,
  memoryManager,
  mapOutputTracker,
  shuffleManager,
  blockTransferService,
  securityManager,
  externalShuffleClient)
```

