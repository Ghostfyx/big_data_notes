# Dispatcher 源码分析

## 1. 简介

Dispatcher消息分发程序，负责将RPC消息路由到适当的端点。

## 2. 源码解析

### 2.1 主要属性

```scala
//每个 RpcEndpoint 其实都会被整合成一个 EndpointData 。并且每个 RpcEndpoint 都会有一个 inbox。
private class EndpointData(
    val name: String,
    val endpoint: RpcEndpoint,
    val ref: NettyRpcEndpointRef) {
    val inbox = new Inbox(ref, endpoint)
}

// Dispatcher内部维护一个线程安全的endpoint和endpointRef集合
private val endpoints: ConcurrentMap[String, EndpointData] =
new ConcurrentHashMap[String, EndpointData]
private val endpointRefs: ConcurrentMap[RpcEndpoint, RpcEndpointRef] =
new ConcurrentHashMap[RpcEndpoint, RpcEndpointRef]

// Track the receivers whose inboxes may contain messages.
//一个阻塞队列，当有 RpcEndpoint 相关请求（InboxMessage）的时候，就会将请求塞到这个队列中，然后被线程池处理。
private val receivers = new LinkedBlockingQueue[EndpointData]

/**
   * True if the dispatcher has been stopped. Once stopped, all messages posted will be bounced
   * immediately.
   */
@GuardedBy("this")
private var stopped = false
```

### 2.2.2 主要方法

**registerRpcEndpoint**

```scala
/**
    * 将 RpcEndpoint 注册到 Dispatcher中，当有 Message 到来的时候，便会分发 Message 到相应的 RpcEndPoint 中进行处理
    * @param name  RpcEndpoint名称
    * @param endpoint RpcEndpoint终端点
    * @return
    */
def registerRpcEndpoint(name: String, endpoint: RpcEndpoint): NettyRpcEndpointRef = {
    val addr = RpcEndpointAddress(nettyEnv.address, name)
    val endpointRef = new NettyRpcEndpointRef(nettyEnv.conf, addr, nettyEnv)
    // 加锁，线程安全
    synchronized {
        // 多线程环境下，注册一个 RpcEndpoint 需要判断现在是否处于 stop 状态。
        if (stopped) {
            throw new IllegalStateException("RpcEnv has been stopped")
        }
        if (endpoints.putIfAbsent(name, new EndpointData(name, endpoint, endpointRef)) != null) {
            throw new IllegalArgumentException(s"There is already an RpcEndpoint called $name")
        }
        val data = endpoints.get(name)
        endpointRefs.put(data.endpoint, data.ref)
        //将 这个 EndpointData 加入到 receivers 队列中，此时 dispatcher 中的 threadpool 会去处理这个加进来的 EndpointData
        //处理过程是调用它的 inbox 的 process()方法。然后 inbox 会等待消息到来。
        receivers.offer(data)  // for the OnStart message
    }
    endpointRef
}
```

**postMessage**

基于postMessage方法，Dispatcher提供了四种发送方法：postRemoteMessage、postLocalMessage、postToAll与postOneWayMessage。

```scala
/** Posts a message sent by a remote endpoint. */
// 关键在于message类型不同

/**
   * Send a message to all registered [[RpcEndpoint]]s in this process.
   *
   * This can be used to make network events known to all end points (e.g. "a new node connected").
   */
def postToAll(message: InboxMessage): Unit = {
    val iter = endpoints.keySet().iterator()
    while (iter.hasNext) {
        val name = iter.next
        postMessage(name, message, (e) => { e match {
            case e: RpcEnvStoppedException => logDebug (s"Message $message dropped. ${e.getMessage}")
            case e: Throwable => logWarning(s"Message $message dropped. ${e.getMessage}")
        }}
                   )}
}

def postRemoteMessage(message: RequestMessage, callback: RpcResponseCallback): Unit = {
  val rpcCallContext =
    new RemoteNettyRpcCallContext(nettyEnv, callback, message.senderAddress)
  val rpcMessage = RpcMessage(message.senderAddress, message.content, rpcCallContext)
  postMessage(message.receiver.name, rpcMessage, (e) => callback.onFailure(e))
}

/** Posts a message sent by a local endpoint. */
def postLocalMessage(message: RequestMessage, p: Promise[Any]): Unit = {
  val rpcCallContext =
    new LocalNettyRpcCallContext(message.senderAddress, p)
  val rpcMessage = RpcMessage(message.senderAddress, message.content, rpcCallContext)
  postMessage(message.receiver.name, rpcMessage, (e) => p.tryFailure(e))
}

/** Posts a one-way message. */
def postOneWayMessage(message: RequestMessage): Unit = {
  postMessage(message.receiver.name, OneWayMessage(message.senderAddress, message.content),
    (e) => throw e)
}
```

```scala
/**
   * Posts a message to a specific endpoint.
   * 向指定RpcEndpoint发送消息
   * @param endpointName name of the endpoint.
   * @param message the message to post
   * @param callbackIfStopped callback function if the endpoint is stopped.
   */
  private def postMessage(
      endpointName: String,
      message: InboxMessage,
      callbackIfStopped: (Exception) => Unit): Unit = {
    val error = synchronized {
      val data = endpoints.get(endpointName)
      if (stopped) {
        Some(new RpcEnvStoppedException())
      } else if (data == null) {
        Some(new SparkException(s"Could not find $endpointName."))
      } else {
        // 发送消息到指定endpoint的inbox
        data.inbox.post(message)
        // 插入接收队列的尾部
        receivers.offer(data)
        None
      }
    }
    // We don't need to call `onStop` in the `synchronized` block
    error.foreach(callbackIfStopped)
  }
```

