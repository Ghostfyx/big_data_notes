# Client源码分析

## 1. run()

YarnClusterApplication类中运行的是Client中run方法，Client#run()中实现了任务提交流程。

```scala
/**
   * 向YARN的ResourceManager提交application
   * 如果设置了spark.yarn.submit.waitAppCompletion=true，它会一直保持活动状态，报告应用程序的状态，直到应用程序由于任何原因退出。
   * 否则，客户端会在提交应用后退出。如果应用程序以失败，中止或未定义状态结束，抛出适当的SparkException。
   *
   * Submit an application to the ResourceManager.
   * If set spark.yarn.submit.waitAppCompletion to true, it will stay alive
   * reporting the application's status until the application has exited for any reason.
   * Otherwise, the client process will exit after submission.
   * If the application finishes with a failed, killed, or undefined status,
   * throw an appropriate SparkException.
   */
def run(): Unit = {
    // 提交应用
    this.appId = submitApplication()
    if (!launcherBackend.isConnected() && fireAndForget) {
        val report = getApplicationReport(appId)
        val state = report.getYarnApplicationState
        logInfo(s"Application report for $appId (state: $state)")
        logInfo(formatReportDetails(report))
        if (state == YarnApplicationState.FAILED || state == YarnApplicationState.KILLED) {
            throw new SparkException(s"Application $appId finished with status: $state")
        }
    } else {
        // 监控YARN中Application状态
        val YarnAppReport(appState, finalState, diags) = monitorApplication(appId)
        if (appState == YarnApplicationState.FAILED || finalState == FinalApplicationStatus.FAILED) {
            diags.foreach { err =>
                logError(s"Application diagnostics message: $err")
            }
            throw new SparkException(s"Application $appId finished with failed status")
        }
        if (appState == YarnApplicationState.KILLED || finalState == FinalApplicationStatus.KILLED) {
            throw new SparkException(s"Application $appId is killed")
        }
        if (finalState == FinalApplicationStatus.UNDEFINED) {
            throw new SparkException(s"The final status of application $appId is undefined")
        }
    }
}
```

## 2. submitApplication()

内部调用org.apache.hadoop.yarn.client.api.{YarnClient, YarnClientApplication}创建yarnClient，向RM注册AM并申请运行AM所需资源，

```scala
/**
   * Submit an application running our ApplicationMaster to the ResourceManager.
   * 将运行pplicationMaster的应用程序提交到ResourceManager。
   *
   * The stable Yarn API provides a convenience method (YarnClient#createApplication) for
   * creating applications and setting up the application submission context. This was not
   * available in the alpha API.
   */
def submitApplication(): ApplicationId = {
    var appId: ApplicationId = null
    try {
        launcherBackend.connect()
        // 初始化yarnClient
        yarnClient.init(hadoopConf)
        // yarnClient开启
        yarnClient.start()

        logInfo("Requesting a new application from cluster with %d NodeManagers"
                .format(yarnClient.getYarnClusterMetrics.getNumNodeManagers))

        // Get a new application from our RM
        // 在YARN集群的RM上创建Application,该newApp用于运行AM
        val newApp = yarnClient.createApplication()
        // 返回newApp需要资源情况(newAppResponse)
        val newAppResponse = newApp.getNewApplicationResponse()
        appId = newAppResponse.getApplicationId()

        new CallerContext("CLIENT", sparkConf.get(APP_CALLER_CONTEXT),
                          Option(appId.toString)).setCurrentContext()

        // Verify whether the cluster has enough resources for our AM
        // 验证集群是否有足够的资源来运行AM
        verifyClusterResources(newAppResponse)

        // Set up the appropriate contexts to launch our AM
        // 关键是这两个方法:
        // 1. 创建ApplicationMaster ContainerLaunch上下文，将ContainerLaunch命令、jar包、java变量等环境准备完毕；
        // 2. 创建Application提交至YARN的上下文，主要读取配置文件设置调用YARN接口前的上下文变量。
        val containerContext = createContainerLaunchContext(newAppResponse)
        val appContext = createApplicationSubmissionContext(newApp, containerContext)

        // Finally, submit and monitor the application
        logInfo(s"Submitting application $appId to ResourceManager")
        // 调用yarnClient#submitApplication(appContext)向yarn提交任务启动的请求，并监控application
        yarnClient.submitApplication(appContext)
        launcherBackend.setAppId(appId.toString)
        reportLauncherState(SparkAppHandle.State.SUBMITTED)

        appId
    } catch {
        case e: Throwable =>
        if (appId != null) {
            cleanupStagingDir(appId)
        }
        throw e
    }
}
```

