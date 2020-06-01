# Worker源码分析

## 1. Worker启动源码分析

Spark Worker节点启动脚本start-slaves.sh脚本底层调用start-slave.sh，start-slave.sh 底层调用

```sh
 "${SPARK_HOME}/sbin"/spark-daemon.sh start $CLASS $WORKER_NUM \
     --webui-port "$WEBUI_PORT" $PORT_FLAG $PORT_NUM $MASTER "$@"
```

CLASS的值为org.apache.spark.deploy.worker.Worker。Worker启动时调用Worker类的main方法，在Worker节点所在机器启动JVM。Worker.scala的main方法源码如下：

```scala
def main(argStrings: Array[String]) {
    // 设置Spark未捕获异常的处理器
    Thread.setDefaultUncaughtExceptionHandler(new SparkUncaughtExceptionHandler(
      exitOnUncaughtException = false))
    Utils.initDaemon(log)
    val conf = new SparkConf
    val args = new WorkerArguments(argStrings, conf)
    // 创建、开启Worker RpcEznv和RpcEndPoint
    val rpcEnv = startRpcEnvAndEndpoint(args.host, args.port, args.webUiPort, args.cores,
      args.memory, args.masters, args.workDir, conf = conf)
    // With external shuffle service enabled, if we request to launch multiple workers on one host,
    // we can only successfully launch the first worker and the rest fails, because with the port
    // bound, we may launch no more than one external shuffle service on each host.
    // When this happens, we should give explicit reason of failure instead of fail silently. For
    // more detail see SPARK-20989.

    // 是否启用外部shuffle服务，如果我们请求在一个主机上启动多个工作程序，则我们只能成功启动第一个工作程序，而其他程序则失败，
    // 因为在绑定端口的情况下，我们在每个主机上最多可以启动一个external Shuffle Service
    val externalShuffleServiceEnabled = conf.get(config.SHUFFLE_SERVICE_ENABLED)
    // 获取Spark启动时设置的Worker数量
    val sparkWorkerInstances = scala.sys.env.getOrElse("SPARK_WORKER_INSTANCES", "1").toInt
    require(externalShuffleServiceEnabled == false || sparkWorkerInstances <= 1,
      "Starting multiple workers on one host is failed because we may launch no more than one " +
        "external shuffle service on each host, please set spark.shuffle.service.enabled to " +
        "false or set SPARK_WORKER_INSTANCES to 1 to resolve the conflict.")
    rpcEnv.awaitTermination()
  }

```

startRpcEnvAndEndpoint 源码如下：

```scala
def startRpcEnvAndEndpoint(
      host: String,
      port: Int,
      webUiPort: Int,
      cores: Int,
      memory: Int,
      masterUrls: Array[String],
      workDir: String,
      workerNumber: Option[Int] = None,
      conf: SparkConf = new SparkConf): RpcEnv = {

    // The LocalSparkCluster runs multiple local sparkWorkerX RPC Environments
    val systemName = SYSTEM_NAME + workerNumber.map(_.toString).getOrElse("")
    val securityMgr = new SecurityManager(conf)
    val rpcEnv = RpcEnv.create(systemName, host, port, conf, securityMgr)
    // new Worker新建Worker实例，调用worker onstart方法
    val masterAddresses = masterUrls.map(RpcAddress.fromSparkURL(_))
    rpcEnv.setupEndpoint(ENDPOINT_NAME, new Worker(rpcEnv, webUiPort, cores, memory,
      masterAddresses, ENDPOINT_NAME, workDir, conf, securityMgr))
    rpcEnv
  }
```

## 1.1 onStart源码

```scala
override def onStart() {
    assert(!registered)
    logInfo("Starting Spark worker %s:%d with %d cores, %s RAM".format(
      host, port, cores, Utils.megabytesToString(memory)))
    logInfo(s"Running Spark version ${org.apache.spark.SPARK_VERSION}")
    logInfo("Spark home: " + sparkHome)
    // 创建Worker工作目录
    createWorkDir()
    startExternalShuffleService()
    webUi = new WorkerWebUI(this, workDir, webUiPort)
    webUi.bind()

    workerWebUiUrl = s"http://$publicAddress:${webUi.boundPort}"
    // 向master注册Worker
    registerWithMaster()
    // 向Spark指标体系注册Worker
    metricsSystem.registerSource(workerSource)
    metricsSystem.start()
    // Attach the worker metrics servlet handler to the web ui after the metrics system is started.
    metricsSystem.getServletHandlers.foreach(webUi.attachHandler)
  }

// 向master注册worker
private def registerWithMaster() {
    // onDisconnected may be triggered multiple times, so don't attempt registration
    // if there are outstanding registration attempts scheduled.
    registrationRetryTimer match {
      case None =>
        registered = false
        // HA情况下向所有Master注册
        registerMasterFutures = tryRegisterAllMasters()
        connectionAttemptCount = 0
        registrationRetryTimer = Some(forwordMessageScheduler.scheduleAtFixedRate(
          new Runnable {
            override def run(): Unit = Utils.tryLogNonFatalError {
              Option(self).foreach(_.send(ReregisterWithMaster))
            }
          },
          INITIAL_REGISTRATION_RETRY_INTERVAL_SECONDS,
          INITIAL_REGISTRATION_RETRY_INTERVAL_SECONDS,
          TimeUnit.SECONDS))
      case Some(_) =>
        logInfo("Not spawning another attempt to register with the master, since there is an" +
          " attempt scheduled already.")
    }
  }
```

