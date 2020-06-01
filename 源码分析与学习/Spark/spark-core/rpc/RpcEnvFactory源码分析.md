# RpcEnvFactory源码分析

## 1. 作用

RpcEndpoint和RpcEndpointRef有一个管理者：RpcEnv。

RpcEnv是一个RpcEndpoints用于处理消息的环境，管理着整个RpcEndpoint的生命周期：（1）根据name或uri注册endpoints；（2）管理各种消息的处理；（3）停止endpoints。RpcEnv必须通过工厂类RpcEnvFactory创建。

RpcEnv的工厂方法，用户创建RpcEnv，Spark 2.0之后使用Netty作为Rpc通信框架，因此唯一实现类为`NettyRpcEnvFactory`。

## 2. 源码

```scala
/**
 * A factory class to create the [[RpcEnv]]. It must have an empty constructor so that it can be
 * created using Reflection.
 */
private[spark] trait RpcEnvFactory {

  def create(config: RpcEnvConfig): RpcEnv
}
```

## 3. NettyRpcEnvFactory创建RpcEnv源码

```scala
private[rpc] class NettyRpcEnvFactory extends RpcEnvFactory with Logging {

  def create(config: RpcEnvConfig): RpcEnv = {
    val sparkConf = config.conf
    // Use JavaSerializerInstance in multiple threads is safe. However, if we plan to support
    // KryoSerializer in future, we have to use ThreadLocal to store SerializerInstance
    val javaSerializerInstance =
      new JavaSerializer(sparkConf).newInstance().asInstanceOf[JavaSerializerInstance]
    val nettyEnv =
      new NettyRpcEnv(sparkConf, javaSerializerInstance, config.advertiseAddress,
        config.securityManager, config.numUsableCores)
    // 如果不是客户端模式，则以集群模式启动address:port
    if (!config.clientMode) {
      val startNettyRpcEnv: Int => (NettyRpcEnv, Int) = { actualPort =>
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

