# RpcEndpoint源码分析

## 1. RpcEndpoint简介

Spark不同组件之间是通过RPC（Remote Procedure Call) 进行点对点通信的。Spark 1.6推出了RPCEnv，RPCEndpoint，RPCEndpointRef为核心的新型架构下的RPC通信方式。

RpcEndpoint为Rpc通信的终端点。RPCEndpointRef为RpcEndpoint的引用，通过调用RPCEndpointRef向对应的RpcEndpoint发送消息。

## 2. RpcEndPoint源码分析

```scala
/**
 * An end point for the RPC that defines what functions to trigger given a message.
 * RPC的端点，它定义给定消息触发什么功能。
 * It is guaranteed that `onStart`, `receive` and `onStop` will be called in sequence.
 * 可以确保依次调用onStart，receive和onStop。
 * The life-cycle of an endpoint is:
 * endPoint的生命周期是：constructor -> onStart -> receive* -> onStop
 * {@code constructor -> onStart -> receive* -> onStop}
 * 注意：可以多线程同事调用receive方法，如果需要线程安全的receive方法，请使用ThreadSafeRpcEndpoint类
 * Note: `receive` can be called concurrently. If you want `receive` to be thread-safe, please use
 * [[ThreadSafeRpcEndpoint]]
 * 如果RpcEndpoint类抛出任何异常，onError方法就会本调用，如果onError方法抛出异常，RpcEnv会忽略这个异常
 * If any error is thrown from one of [[RpcEndpoint]] methods except `onError`, `onError` will be
 * invoked with the cause. If `onError` throws an error, [[RpcEnv]] will ignore it.
 */
private[spark] trait RpcEndpoint {

  /**
   * The [[RpcEnv]] that this [[RpcEndpoint]] is registered to.
   */
  val rpcEnv: RpcEnv

  /**
   * The [[RpcEndpointRef]] of this [[RpcEndpoint]]. `self` will become valid when `onStart` is
   * called. And `self` will become `null` when `onStop` is called.
   *
   * Note: Because before `onStart`, [[RpcEndpoint]] has not yet been registered and there is not
   * valid [[RpcEndpointRef]] for it. So don't call `self` before `onStart` is called.
   */
  final def self: RpcEndpointRef = {
    require(rpcEnv != null, "rpcEnv has not been initialized")
    rpcEnv.endpointRef(this)
  }

  /**
   * Process messages from `RpcEndpointRef.send` or `RpcCallContext.reply`. If receiving a
   * unmatched message, `SparkException` will be thrown and sent to `onError`.
    * 处理调用RpcEndpointRef.send和RpcCallContext.reply方法发出的消息
   */
  def receive: PartialFunction[Any, Unit] = {
    case _ => throw new SparkException(self + " does not implement 'receive'")
  }

  /**
   * Process messages from `RpcEndpointRef.ask`. If receiving a unmatched message,
   * `SparkException` will be thrown and sent to `onError`.
    * 处理调用RpcEndpointRef.ask发出的消息，并回复消息
   */
  def receiveAndReply(context: RpcCallContext): PartialFunction[Any, Unit] = {
    case _ => context.sendFailure(new SparkException(self + " won't reply anything"))
  }

  /**
   * Invoked when any exception is thrown during handling messages.
   */
  def onError(cause: Throwable): Unit = {
    // By default, throw e and let RpcEnv handle it
    throw cause
  }

  /**
   * Invoked when `remoteAddress` is connected to the current node.
   */
  def onConnected(remoteAddress: RpcAddress): Unit = {
    // By default, do nothing.
  }

  /**
   * Invoked when `remoteAddress` is lost.
   */
  def onDisconnected(remoteAddress: RpcAddress): Unit = {
    // By default, do nothing.
  }

  /**
   * Invoked when some network error happens in the connection between the current node and
   * `remoteAddress`.
   */
  def onNetworkError(cause: Throwable, remoteAddress: RpcAddress): Unit = {
    // By default, do nothing.
  }

  /**
   * Invoked before [[RpcEndpoint]] starts to handle any message.
    * 在RpcEndpoint开始处理消息之间就调用
    *
   */
  def onStart(): Unit = {
    // By default, do nothing.
  }

  /**
   * Invoked when [[RpcEndpoint]] is stopping. `self` will be `null` in this method and you cannot
   * use it to send or ask messages.
    * 在RpcEndpoint停止时调用
   */
  def onStop(): Unit = {
    // By default, do nothing.
  }

  /**
   * A convenient method to stop [[RpcEndpoint]].
   */
  final def stop(): Unit = {
    val _self = self
    if (_self != null) {
      rpcEnv.stop(_self)
    }
  }
}
```

