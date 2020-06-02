# NettyRpcEnv源码分析

## 1. object NettyRpcEnv

```scala
private[netty] object NettyRpcEnv extends Logging {
  /**
   * When deserializing the [[NettyRpcEndpointRef]], it needs a reference to [[NettyRpcEnv]].
   * Use `currentEnv` to wrap the deserialization codes. E.g.,
   *
   * 反序列化[[NettyRpcEndpointRef]]时，需要引用[[NettyRpcEnv]]。
   * {{{
   *   NettyRpcEnv.currentEnv.withValue(this) {
   *     your deserialization codes
   *   }
   * }}}
   */
  private[netty] val currentEnv = new DynamicVariable[NettyRpcEnv](null)

  /**
   * Similar to `currentEnv`, this variable references the client instance associated with an
   * RPC, in case it's needed to find out the remote address during deserialization.
   * 与currentEnv相似，此变量引用与PRC关联的客户端实例，以防在反序列化期间需要找到远程地址。
   */
  private[netty] val currentClient = new DynamicVariable[TransportClient](null)

}
```

## 2. class NettyRpcEnv

```scala
private[netty] class NettyRpcEnv(
    val conf: SparkConf,
    javaSerializerInstance: JavaSerializerInstance,
    host: String,
    securityManager: SecurityManager,
    numUsableCores: Int) extends RpcEnv(conf) with Logging {

  private[netty] val transportConf = SparkTransportConf.fromSparkConf(
    conf.clone.set("spark.rpc.io.numConnectionsPerPeer", "1"),
    "rpc",
    conf.getInt("spark.rpc.io.threads", numUsableCores))

  // Rpc请求转发器
  private val dispatcher: Dispatcher = new Dispatcher(this, numUsableCores)

  private val streamManager = new NettyStreamManager(this)
  //TransportContext 属于 spark.network 中的部分，负责 RPC 消息在网络中的传输
  private val transportContext = new TransportContext(transportConf,
    new NettyRpcHandler(dispatcher, this, streamManager))
}
```

以下为重点介绍的几个方法：

### 2.1 startServer

启动服务端的RpcEnv服务。

```scala
def startServer(bindAddress: String, port: Int): Unit = {
    val bootstraps: java.util.List[TransportServerBootstrap] =
      if (securityManager.isAuthenticationEnabled()) {
        java.util.Arrays.asList(new AuthServerBootstrap(transportConf, securityManager))
      } else {
        java.util.Collections.emptyList()
      }
    //TransportContext 属于 spark.network 中的部分，负责 RPC 消息在网络中的传输
    server = transportContext.createServer(bindAddress, port, bootstraps)
    // 在每个 RpcEndpoint 注册的时候都会注册一个默认的 RpcEndpointVerifier，它的作用是客户端调用的时候先用它来询问 Endpoint 是否存在。
    dispatcher.registerRpcEndpoint(
      RpcEndpointVerifier.NAME, new RpcEndpointVerifier(this, dispatcher))
  }
```

### 2.2 setupEndpoint

调用 dispatcher注册Endpoint。

```scala
  override def setupEndpoint(name: String, endpoint: RpcEndpoint): RpcEndpointRef = {
      // 详细见dispatcher源码分析
    dispatcher.registerRpcEndpoint(name, endpoint)
  }
```

### 2.3 send

```scala
/**
    * 发送Rpc消息
    * @param message
    */
private[netty] def send(message: RequestMessage): Unit = {
    // 远端通信IP地址
    val remoteAddr = message.receiver.address
    if (remoteAddr == address) {
        // Message to a local RPC endpoint.
        try {
            // 如果本地发送给自己则直接转发
            dispatcher.postOneWayMessage(message)
        } catch {
            case e: RpcEnvStoppedException => logDebug(e.getMessage)
        }
    } else {
        // Message to a remote RPC endpoint.
        // 将消息放入Outbox
        postToOutbox(message.receiver, OneWayOutboxMessage(message.serialize(this)))
    }
}
```

### 2.4 endpointRef

从dispatcher获取终端点的引用。

```scala
override def endpointRef(endpoint: RpcEndpoint): RpcEndpointRef = {
    dispatcher.getRpcEndpointRef(endpoint)
}
```

