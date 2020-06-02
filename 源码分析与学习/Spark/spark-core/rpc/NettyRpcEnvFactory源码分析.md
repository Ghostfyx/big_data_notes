# NettyRpcEnvFactory源码分析

## 1. 简介

NettyRpcEnvFactory继承RpcEnvFactory类，重写了create方法，用于创建RpcEnv。

## 2. 源码

```scala
private[rpc] class NettyRpcEnvFactory extends RpcEnvFactory with Logging {

  def create(config: RpcEnvConfig): RpcEnv = {
    val sparkConf = config.conf
    // Use JavaSerializerInstance in multiple threads is safe. However, if we plan to support
    // KryoSerializer in future, we have to use ThreadLocal to store SerializerInstance
    // 实例化序列化类，使用JavaSerializerInstance是线程安全的，未来会提供KryoSerializer，但是必须使用ThreadLocal存储KryoSerializer实例
    val javaSerializerInstance =
      new JavaSerializer(sparkConf).newInstance().asInstanceOf[JavaSerializerInstance]
    // 实例化nettyEnv对象
    val nettyEnv =
      new NettyRpcEnv(sparkConf, javaSerializerInstance, config.advertiseAddress,
        config.securityManager, config.numUsableCores)
    //如果是服务端创建的，那么会启动服务。服务端和客户端都会通过这个方法创建一个 NettyRpcEnv ，但区别就在这里了。
    if (!config.clientMode) {
      val startNettyRpcEnv: Int => (NettyRpcEnv, Int) = { actualPort =>
        //启动服务的方法,下一步就是调用这个方法了
        nettyEnv.startServer(config.bindAddress, actualPort)
        (nettyEnv, nettyEnv.address.port)
      }
      try {
        Utils.startServiceOnPort(config.port, startNettyRpcEnv, sparkConf, config.name)._1
      } catch {
        case NonFatal(e) =>
          nettyEnv.shutdown()
          throw e
      }
    }
    nettyEnv
  }
}
```

该方法主要目的是创建NettyRpcEnv，并启动对应的服务。具体参见NettyRpcEnv源码分析。