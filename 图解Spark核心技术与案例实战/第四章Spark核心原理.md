# 第四章 Spark核心原理

在描述Spark运行基本流程前，我们先介绍Spark基本概念，如图4-1所示：

![](http://spark.apache.org/docs/latest/img/cluster-overview.png)

​																		**图4-1 Spark运行结构图**

- **Applocation(应用程序)：**是指用户编写的Spark应用程序，包含驱动程序(Driver)和分布在集群多个节点上运行的Executor代码，在执行过程中由一到多个作业组成。
- **Driver( 驱动程序)：**Spark中的Driver即运行上述Application的main函数并创建SparkContext，其中创建SparkContext的目的是为了准备Spark Application的运行环境。SparkContext负责与ClusterManager通信，进行资源申请、任务分配和监控等。当Executor执行完毕，Driver负责将SparkContext关闭。
- **Cluster Manager(集群资源管理器)：**是指在集群上获取资源的外部服务，目前有以下几种资源管理器：
	1. **Standalone：**Spark原生资源管理，由Master节点负责资源管理
	2. **Hadoop Yarn：**由Yarn的ResourceManager负责资源的管理
	3. **Mesos：**由Mesos中的Mesos Master负责管理
- **Worker(工作节点)：**集群中任何可以运行Application代码的节点。在Yarn和Spark on Yarn模式中指的是NodeManager；在Standalone中指的是通过Slave文件配置的Worker节点。
- **Master(总控进程)：**Spark Standalone模式下的主节点，负责管理和分配集群资源来运行Spark Application。
- **Executor(执行进程)：**Aplication运行在Worker节点上的一个进程，该进程负责从线程池中抽取空余线程运行Task，Task并行的数量取决于分配给Executor进程的CPU数量。最终Executor将结果写入内存或磁盘中，每个Application都独立拥有一批Executor。

## 4.1 消息通信原理

### 4.1.1 Spark消息通信架构

Spark中定义了通信框架接口，接口实现中调用Netty的具体方法（Spark 2.0版本之后）。在框架中以RpcEndPoint和RpcEndPointRef实现了Actor和ActorRef相关动作，RpcEndPointRef是RpcEndPoint的引用，它们关系如图4-2所示：

![](./img/4-2.jpg)

​																	**图4-2 Spark消息通讯类图**

消息通讯框架使用工厂模式，实现了对Netty的结偶，能够根据需要引入其他消息通讯工具。Spark具体实现步骤如下：

- 定义RpcEnvFactory与RpcEnv两个抽象类，类的描述如图4-3，4-4所示。RpcEnv定义了RPC通信框架启动、关闭和停止等抽象方法，RpcEnvFactory定义了创建抽象方法；

	<img src="./img/4-3.jpg" style="zoom:60%;" />

	​															**图4-3 RpcEnvFactory**

	<img src="./img/4-4.jpg" style="zoom:50%;" />

	​																	**图4-4 RpcEnv**

- NettyRpcEnv和NettyRpcFactory使用Netty对继承方法进行了实现，NettyRpcEnv中的setupEndPoint方法将RpcEndpoint和RpcEndpointRef已键值对的方式存放在线程安全的ConcurrentHashMap中。

- 在RpcEnv类中通过反射的方式实现了创建RpcEnv实例的静态方法。

在各模块使用中，如Master、Worker等，会先使用RpcEnv的静态方法创建RpcEnv实例，然后实例化Master，由于Master继承于ThreadSafeRpcEndpoint，因此创建的Master实例是一个线程安全的终端点，接着调用RpcEnv的setupEndPoint方法，把Master的终端点和其对应的引用注册到RpcEnv中。在消息通信中，只要其他对象获取了Master终端点的引用，就可以与Master通信。

Master.scala类的startRpcEnvAndEndPoint方法启动消息通信框架代码如下：

```scala
/**
   * Start the Master and return a three tuple of:
   *   (1) The Master RpcEnv
   *   (2) The web UI bound port
   *   (3) The REST server bound port, if any
   */
  def startRpcEnvAndEndpoint(
      host: String,
      port: Int,
      webUiPort: Int,
      conf: SparkConf): (RpcEnv, Int, Option[Int]) = {
    val securityMgr = new SecurityManager(conf)
    val rpcEnv = RpcEnv.create(SYSTEM_NAME, host, port, conf, securityMgr)
    val masterEndpoint = rpcEnv.setupEndpoint(ENDPOINT_NAME,
      new Master(rpcEnv, rpcEnv.address, webUiPort, securityMgr, conf))
    val portsResponse = masterEndpoint.askSync[BoundPortsResponse](BoundPortsRequest)
    (rpcEnv, portsResponse.webUIPort, portsResponse.restPort)
  }
```

Spark运行过程中，Master、Driver、Worker以及Executor等模块之间由实践驱动消息的发送。下面以Standalone为例，分析Spark启动过程和Application运行过程中如何通信。