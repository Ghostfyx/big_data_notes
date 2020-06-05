# SparkSubmit源码分析

## 1. object SparkSubmit

**属性**

```scala
 // Cluster managers 集群资源管理器
  private val YARN = 1
  private val STANDALONE = 2
  private val MESOS = 4
  private val LOCAL = 8
  private val KUBERNETES = 16
  private val ALL_CLUSTER_MGRS = YARN | STANDALONE | MESOS | LOCAL | KUBERNETES

  // Deploy modes 部署模式：客户端模式，集群模式
  private val CLIENT = 1
  private val CLUSTER = 2
  private val ALL_DEPLOY_MODES = CLIENT | CLUSTER

  // Special primary resource names that represent shells rather than application jars.
  private val SPARK_SHELL = "spark-shell"
  private val PYSPARK_SHELL = "pyspark-shell"
  private val SPARKR_SHELL = "sparkr-shell"
  private val SPARKR_PACKAGE_ARCHIVE = "sparkr.zip"
  private val R_PACKAGE_ARCHIVE = "rpkg.zip"

  private val CLASS_NOT_FOUND_EXIT_STATUS = 101

  // Following constants are visible for testing.
  private[deploy] val YARN_CLUSTER_SUBMIT_CLASS =
    "org.apache.spark.deploy.yarn.YarnClusterApplication"
  private[deploy] val REST_CLUSTER_SUBMIT_CLASS = classOf[RestSubmissionClientApp].getName()
  private[deploy] val STANDALONE_CLUSTER_SUBMIT_CLASS = classOf[ClientApp].getName()
  private[deploy] val KUBERNETES_CLUSTER_SUBMIT_CLASS =
    "org.apache.spark.deploy.k8s.submit.KubernetesClientApplication"
```

**Main方法**

SparkSubmit类的main方法是Spark-submit脚本的入口，源码如下：

```scala
override def main(args: Array[String]): Unit = {
    // 新建SparkSubmit类，并重写部分方法
    val submit = new SparkSubmit() {
      self =>

      // 参数组织与整理
      override protected def parseArguments(args: Array[String]): SparkSubmitArguments = {
        new SparkSubmitArguments(args) {
          override protected def logInfo(msg: => String): Unit = self.logInfo(msg)

          override protected def logWarning(msg: => String): Unit = self.logWarning(msg)
        }
      }

      override protected def logInfo(msg: => String): Unit = printMessage(msg)

      override protected def logWarning(msg: => String): Unit = printMessage(s"Warning: $msg")

      // 提交Application
      override def doSubmit(args: Array[String]): Unit = {
        try {
          // 
          super.doSubmit(args)
        } catch {
          case e: SparkUserAppException =>
            exitFn(e.exitCode)
        }
      }

    }

    submit.doSubmit(args)
  }
```

**其他方法**

用于判断Spark submit中部分属性

```scala
/**
   * Return whether the given primary resource represents a user jar.
   */
private[deploy] def isUserJar(res: String): Boolean = {
    !isShell(res) && !isPython(res) && !isInternal(res) && !isR(res)
}

/**
   * Return whether the given primary resource represents a shell.
   */
private[deploy] def isShell(res: String): Boolean = {
    (res == SPARK_SHELL || res == PYSPARK_SHELL || res == SPARKR_SHELL)
}

/**
   * Return whether the given main class represents a sql shell.
   */
private[deploy] def isSqlShell(mainClass: String): Boolean = {
    mainClass == "org.apache.spark.sql.hive.thriftserver.SparkSQLCLIDriver"
}

/**
   * Return whether the given main class represents a thrift server.
   */
private def isThriftServer(mainClass: String): Boolean = {
    mainClass == "org.apache.spark.sql.hive.thriftserver.HiveThriftServer2"
}

/**
   * Return whether the given primary resource requires running python.
   */
private[deploy] def isPython(res: String): Boolean = {
    res != null && res.endsWith(".py") || res == PYSPARK_SHELL
}

/**
   * Return whether the given primary resource requires running R.
   */
private[deploy] def isR(res: String): Boolean = {
    res != null && res.endsWith(".R") || res == SPARKR_SHELL
}

private[deploy] def isInternal(res: String): Boolean = {
    res == SparkLauncher.NO_RESOURCE
}
```

## 2.  class SparkSubmit

### 2.2.1 doSubmit方法

SparkSubmit对象main方法中新建SparkSubmit类，然后调用其doSubmit方法提交，SparkSubmit类中doSubmit函数是一个主控函数，根据接受的action类型，调用对应的处理：

```scala
def doSubmit(args: Array[String]): Unit = {
    // Initialize logging if it hasn't been done yet. Keep track of whether logging needs to
    // be reset before the application starts.
    val uninitLog = initializeLogIfNecessary(true, silent = true)

    val appArgs = parseArguments(args)
    if (appArgs.verbose) {
      logInfo(appArgs.toString)
    }
    // 根据接受的action类型，调用对应的处理：
    appArgs.action match {
        // 提交Spark任务
      case SparkSubmitAction.SUBMIT => submit(appArgs, uninitLog)
        // kill Spark任务
      case SparkSubmitAction.KILL => kill(appArgs)
        // 获取任务状态
      case SparkSubmitAction.REQUEST_STATUS => requestStatus(appArgs)
        // 打印版本信息
      case SparkSubmitAction.PRINT_VERSION => printVersion()
    }
  }
```

### 2.2.2 submit方法

```scala
/**
   * Submit the application using the provided parameters, ensuring to first wrap
   * in a doAs when --proxy-user is specified.
   */
@tailrec
private def submit(args: SparkSubmitArguments, uninitLog: Boolean): Unit = {

    def doRunMain(): Unit = {
        if (args.proxyUser != null) {
            // 如果是代理用户，则使用proxyUser 对runMain()函数包装调用；
            val proxyUser = UserGroupInformation.createProxyUser(args.proxyUser,                                                       UserGroupInformation.getCurrentUser())
            try {
                proxyUser.doAs(new PrivilegedExceptionAction[Unit]() {
                    override def run(): Unit = {
                        runMain(args, uninitLog)
                    }
                })
            } catch {
                case e: Exception =>
                // Hadoop's AuthorizationException suppresses the exception's stack trace, which
                // makes the message printed to the output by the JVM not very helpful. Instead,
                // detect exceptions with empty stack traces here, and treat them differently.
                if (e.getStackTrace().length == 0) {
                    error(s"ERROR: ${e.getClass().getName()}: ${e.getMessage()}")
                } else {
                    throw e
                }
            }
        } else {
            // 最终实际执行逻辑runMain方法
            runMain(args, uninitLog)
        }
    }

    //在独立集群模式下，有两个提交网关：
    //（1）使用o.a.s.deploy.Client作为包装器的传统RPC网关
    //（2）Spark 1.3中引入了新的基于REST的网关
    //后者是Spark 1.3的默认行为，但如果主端点不是REST服务器，则Spark Submit将故障转移到使用旧网关。
    // In standalone cluster mode, there are two submission gateways:
    //   (1) The traditional RPC gateway using o.a.s.deploy.Client as a wrapper
    //   (2) The new REST-based gateway introduced in Spark 1.3
    // The latter is the default behavior as of Spark 1.3, but Spark submit will fail over
    // to use the legacy gateway if the master endpoint turns out to be not a REST server.
    if (args.isStandaloneCluster && args.useRest) {
        try {
            logInfo("Running Spark using the REST application submission protocol.")
            doRunMain()
        } catch {
            // Fail over to use the legacy submission gateway
            case e: SubmitRestConnectionException =>
            logWarning(s"Master endpoint ${args.master} was not a REST server. " +
                       "Falling back to legacy submission gateway instead.")
            args.useRest = false
            submit(args, false)
        }
        // In all other modes, just run the main class as prepared
    } else {
        doRunMain()
    }
}
```

### 2.2.3 runMain方法

根据用户提交脚本中的参数运行子类中的main方法，关键方法prepareSubmitEnvironment的返回值childArgs, childClasspath, sparkConf与childMainClass。

```scala
/**
   * 使用用户提交脚本中的参数运行子类中的main方法
   *
   * 运行包含两步：
   * 第一步，通过设置适当的类路径，系统属性和应用程序参数来准备启动环境，以便基于集群管理和部署模式运行子主类。
   * 第二步，使用这个启动环境来调用子主类的主方法。
   * Run the main method of the child class using the submit arguments.
   *
   * This runs in two steps. First, we prepare the launch environment by setting up
   * the appropriate classpath, system properties, and application arguments for
   * running the child main class based on the cluster manager and the deploy mode.
   * Second, we use this launch environment to invoke the main method of the child
   * main class.
   *
   * Note that this main class will not be the one provided by the user if we're
   * running cluster deploy mode or python applications.
   */
  private def runMain(args: SparkSubmitArguments, uninitLog: Boolean): Unit = {
    // 通过设置适当的类路径，系统属性和应用程序参数来准备启动环境，以便基于集群管理和部署模式运行子主类
    val (childArgs, childClasspath, sparkConf, childMainClass) = prepareSubmitEnvironment(args)
    // Let the main class re-initialize the logging system once it starts.
    if (uninitLog) {
      Logging.uninitialize()
    }

    if (args.verbose) {
      logInfo(s"Main class:\n$childMainClass")
      logInfo(s"Arguments:\n${childArgs.mkString("\n")}")
      // sysProps may contain sensitive information, so redact before printing
      logInfo(s"Spark config:\n${Utils.redact(sparkConf.getAll.toMap).mkString("\n")}")
      logInfo(s"Classpath elements:\n${childClasspath.mkString("\n")}")
      logInfo("\n")
    }

    val loader =
      if (sparkConf.get(DRIVER_USER_CLASS_PATH_FIRST)) {
        new ChildFirstURLClassLoader(new Array[URL](0),
          Thread.currentThread.getContextClassLoader)
      } else {
        new MutableURLClassLoader(new Array[URL](0),
          Thread.currentThread.getContextClassLoader)
      }
    Thread.currentThread.setContextClassLoader(loader)

    for (jar <- childClasspath) {
      addJarToClasspath(jar, loader)
    }

    var mainClass: Class[_] = null

    try {
       // 通过反射机制获取Application运行主类，底层调用 Class.forName(String name, boolean initialize, ClassLoader loader)
      //  Class.forName(className, true, getContextOrSparkClassLoader) initalize= true，表示给定的类如果之前没有被初始化过，那么会被初始化
      mainClass = Utils.classForName(childMainClass)
    } catch {
      case e: ClassNotFoundException =>
        logWarning(s"Failed to load $childMainClass.", e)
        if (childMainClass.contains("thriftserver")) {
          logInfo(s"Failed to load main class $childMainClass.")
          logInfo("You need to build Spark with -Phive and -Phive-thriftserver.")
        }
        throw new SparkUserAppException(CLASS_NOT_FOUND_EXIT_STATUS)
      case e: NoClassDefFoundError =>
        logWarning(s"Failed to load $childMainClass: ${e.getMessage()}")
        if (e.getMessage.contains("org/apache/hadoop/hive")) {
          logInfo(s"Failed to load hive class.")
          logInfo("You need to build Spark with -Phive and -Phive-thriftserver.")
        }
        throw new SparkUserAppException(CLASS_NOT_FOUND_EXIT_STATUS)
    }

    // 判断SparkApplication(RestSubmissionClientApp, ClientApp)与mainClass是否是同一类型
    val app: SparkApplication = if (classOf[SparkApplication].isAssignableFrom(mainClass)) {  
      mainClass.newInstance().asInstanceOf[SparkApplication]
    } else {
      // SPARK-4170
      if (classOf[scala.App].isAssignableFrom(mainClass)) {
        logWarning("Subclasses of scala.App may not work correctly. Use a main() method instead.")
      }
      new JavaMainApplication(mainClass)
    }

    @tailrec
    def findCause(t: Throwable): Throwable = t match {
      case e: UndeclaredThrowableException =>
        if (e.getCause() != null) findCause(e.getCause()) else e
      case e: InvocationTargetException =>
        if (e.getCause() != null) findCause(e.getCause()) else e
      case e: Throwable =>
        e
    }

    try {
      // 调用Start方法执行提交
      app.start(childArgs.toArray, sparkConf)
    } catch {
      case t: Throwable =>
        throw findCause(t)
    }
  }
```

### 2.2.4 prepareSubmitEnvironment方法

此方法主要是为应用程序准备环境，有几个关键返回值：

- childArgs：子进程的参数
- childClasspath：子级的类路径条目列表
- sparkConf：系统参数map集合
- childMainClass：子级的主类

重点关注了childMainClass(后续提交应用的主类)，childMainClass的取值遵循以下几个条件：

```
1. deployMode == CLIENT，主要是客户端部署模式 childMainClass = --class 
2. StandaloneCluster模式下， spark 1.3 版本之后使用 childMainClass =org.apache.spark.deploy.rest.RestSubmissionClientApp
3. YarnCluster模式下 childMainClass=org.apache.spark.deploy.yarn.YarnClusterApplication
4. MesosCluster模式下，childMainClass =org.apache.spark.deploy.rest.RestSubmissionClientApp
5. KubernetesCluster模式下，childMainClass =org.apache.spark.deploy.k8s.submit.KubernetesClientApplication
```

源码如下：

```scala
/**
   * Prepare the environment for submitting an application.
   * 未提交的应用程序准备环境
   *
   * @param args the parsed SparkSubmitArguments used for environment preparation.
   * @param conf the Hadoop Configuration, this argument will only be set in unit test.
   * @return a 4-tuple:
   *         (1) the arguments for the child process,
   *         (2) a list of classpath entries for the child,
   *         (3) a map of system properties, and
   *         (4) the main class for the child
   *         返回一个4元组(childArgs, childClasspath, sparkConf, childMainClass)
   *         (1) childArgs：子进程的参数
   *         (2) childClasspath：子级的类路径条目列表
   *         (3) sparkConf：系统参数map集合
   *         (4) childMainClass：子级的主类
   *
   *         Exposed for testing.
   */
private[deploy] def prepareSubmitEnvironment(
    args: SparkSubmitArguments,
    conf: Option[HadoopConfiguration] = None)
: (Seq[String], Seq[String], SparkConf, String) = {
    // Return values
    val childArgs = new ArrayBuffer[String]()
    val childClasspath = new ArrayBuffer[String]()
    val sparkConf = new SparkConf()
    var childMainClass = ""

    // Set the cluster manager 根据--master 参数确定资源管理器
    // 设置集群管理器，
    // 从这个列表中可以得到信息：spark目前支持的集群管理器:YARN,STANDLONE,MESOS,KUBERNETES,LOCAL，
    // 在spark-submit参数的--master中指定。
    val clusterManager: Int = args.master match {
        case "yarn" => YARN
        case "yarn-client" | "yarn-cluster" =>
        logWarning(s"Master ${args.master} is deprecated since 2.0." +
                   " Please use master \"yarn\" with specified deploy mode instead.")
        YARN
        case m if m.startsWith("spark") => STANDALONE
        case m if m.startsWith("mesos") => MESOS
        case m if m.startsWith("k8s") => KUBERNETES
        case m if m.startsWith("local") => LOCAL
        case _ =>
        error("Master must either be yarn or start with spark, mesos, k8s, or local")
        -1
    }

    // Set the deploy mode; default is client mode 部署方式：client，cluster
    var deployMode: Int = args.deployMode match {
        // 默认部署方式是client
        case "client" | null => CLIENT
        case "cluster" => CLUSTER
        case _ =>
        error("Deploy mode must be either client or cluster")
        -1
    }

    // 由于”yarn-cluster“和”yarn-client“方式已被弃用，因此封装了--master和--deploy-mode。
    // 如果只指定了一个--master和--deploy-mode，我们有一些逻辑来推断它们之间的关系；如果它们不一致，我们可以提前退出。
    // Because the deprecated way of specifying "yarn-cluster" and "yarn-client" encapsulate both
    // the master and deploy mode, we have some logic to infer the master and deploy mode
    // from each other if only one is specified, or exit early if they are at odds.
    if (clusterManager == YARN) {
        (args.master, args.deployMode) match {
            case ("yarn-cluster", null) =>
            deployMode = CLUSTER
            args.master = "yarn"
            case ("yarn-cluster", "client") =>
            error("Client deploy mode is not compatible with master \"yarn-cluster\"")
            case ("yarn-client", "cluster") =>
            error("Cluster deploy mode is not compatible with master \"yarn-client\"")
            case (_, mode) =>
            args.master = "yarn"
        }

        // Make sure YARN is included in our build if we're trying to use it
        // 确保YARN包含在Spark中
        if (!Utils.classIsLoadable(YARN_CLUSTER_SUBMIT_CLASS) && !Utils.isTesting) {
            error(
                "Could not load YARN classes. " +
                "This copy of Spark may not have been compiled with YARN support.")
        }
    }

    if (clusterManager == KUBERNETES) {
        args.master = Utils.checkAndGetK8sMasterUrl(args.master)
        // Make sure KUBERNETES is included in our build if we're trying to use it
        if (!Utils.classIsLoadable(KUBERNETES_CLUSTER_SUBMIT_CLASS) && !Utils.isTesting) {
            error(
                "Could not load KUBERNETES classes. " +
                "This copy of Spark may not have been compiled with KUBERNETES support.")
        }
    }

    // 以下的一些模式是不支持，尽早让它们失败。
    // Fail fast, the following modes are not supported or applicable
    (clusterManager, deployMode) match {
        case (STANDALONE, CLUSTER) if args.isPython =>
        error("Cluster deploy mode is currently not supported for python " +
              "applications on standalone clusters.")
        case (STANDALONE, CLUSTER) if args.isR =>
        error("Cluster deploy mode is currently not supported for R " +
              "applications on standalone clusters.")
        case (LOCAL, CLUSTER) =>
        error("Cluster deploy mode is not compatible with master \"local\"")
        case (_, CLUSTER) if isShell(args.primaryResource) =>
        error("Cluster deploy mode is not applicable to Spark shells.")
        case (_, CLUSTER) if isSqlShell(args.mainClass) =>
        error("Cluster deploy mode is not applicable to Spark SQL shell.")
        case (_, CLUSTER) if isThriftServer(args.mainClass) =>
        error("Cluster deploy mode is not applicable to Spark Thrift server.")
        case _ =>
    }

    // Update args.deployMode if it is null. It will be passed down as a Spark property later.
    // 如果args.deployMode为null的话，给它赋值更新。稍后它将作为Spark的属性向下传递
    (args.deployMode, deployMode) match {
        case (null, CLIENT) => args.deployMode = "client"
        case (null, CLUSTER) => args.deployMode = "cluster"
        case _ =>
    }
    // 根据资源管理器和部署模式，进行逻辑判断出几种特殊运行方式。
    val isYarnCluster = clusterManager == YARN && deployMode == CLUSTER
    val isMesosCluster = clusterManager == MESOS && deployMode == CLUSTER
    val isStandAloneCluster = clusterManager == STANDALONE && deployMode == CLUSTER
    val isKubernetesCluster = clusterManager == KUBERNETES && deployMode == CLUSTER
    val isMesosClient = clusterManager == MESOS && deployMode == CLIENT

    if (!isMesosCluster && !isStandAloneCluster) {
        // Resolve maven dependencies if there are any and add classpath to jars. Add them to py-files
        // too for packages that include Python code
        val resolvedMavenCoordinates = DependencyUtils.resolveMavenDependencies(
            args.packagesExclusions, args.packages, args.repositories, args.ivyRepoPath,
            args.ivySettingsPath)

        if (!StringUtils.isBlank(resolvedMavenCoordinates)) {
            args.jars = mergeFileLists(args.jars, resolvedMavenCoordinates)
            if (args.isPython || isInternal(args.primaryResource)) {
                args.pyFiles = mergeFileLists(args.pyFiles, resolvedMavenCoordinates)
            }
        }

        // install any R packages that may have been passed through --jars or --packages.
        // Spark Packages may contain R source code inside the jar.
        if (args.isR && !StringUtils.isBlank(args.jars)) {
            RPackageUtils.checkAndBuildRPackage(args.jars, printStream, args.verbose)
        }
    }

    args.sparkProperties.foreach { case (k, v) => sparkConf.set(k, v) }
    val hadoopConf = conf.getOrElse(SparkHadoopUtil.newConfiguration(sparkConf))
    val targetDir = Utils.createTempDir()

    // assure a keytab is available from any place in a JVM
    if (clusterManager == YARN || clusterManager == LOCAL || isMesosClient) {
        if (args.principal != null) {
            if (args.keytab != null) {
                require(new File(args.keytab).exists(), s"Keytab file: ${args.keytab} does not exist")
                // Add keytab and principal configurations in sysProps to make them available
                // for later use; e.g. in spark sql, the isolated class loader used to talk
                // to HiveMetastore will use these settings. They will be set as Java system
                // properties and then loaded by SparkConf
                sparkConf.set(KEYTAB, args.keytab)
                sparkConf.set(PRINCIPAL, args.principal)
                UserGroupInformation.loginUserFromKeytab(args.principal, args.keytab)
            }
        }
    }

    // Resolve glob path for different resources.
    args.jars = Option(args.jars).map(resolveGlobPaths(_, hadoopConf)).orNull
    args.files = Option(args.files).map(resolveGlobPaths(_, hadoopConf)).orNull
    args.pyFiles = Option(args.pyFiles).map(resolveGlobPaths(_, hadoopConf)).orNull
    args.archives = Option(args.archives).map(resolveGlobPaths(_, hadoopConf)).orNull

    lazy val secMgr = new SecurityManager(sparkConf)

    // In client mode, download remote files.
    var localPrimaryResource: String = null
    var localJars: String = null
    var localPyFiles: String = null
    if (deployMode == CLIENT) {
        localPrimaryResource = Option(args.primaryResource).map {
            downloadFile(_, targetDir, sparkConf, hadoopConf, secMgr)
        }.orNull
        localJars = Option(args.jars).map {
            downloadFileList(_, targetDir, sparkConf, hadoopConf, secMgr)
        }.orNull
        localPyFiles = Option(args.pyFiles).map {
            downloadFileList(_, targetDir, sparkConf, hadoopConf, secMgr)
        }.orNull
    }

    // When running in YARN, for some remote resources with scheme:
    //   1. Hadoop FileSystem doesn't support them.
    //   2. We explicitly bypass Hadoop FileSystem with "spark.yarn.dist.forceDownloadSchemes".
    // We will download them to local disk prior to add to YARN's distributed cache.
    // For yarn client mode, since we already download them with above code, so we only need to
    // figure out the local path and replace the remote one.
    if (clusterManager == YARN) {
        val forceDownloadSchemes = sparkConf.get(FORCE_DOWNLOAD_SCHEMES)

        def shouldDownload(scheme: String): Boolean = {
            forceDownloadSchemes.contains("*") || forceDownloadSchemes.contains(scheme) ||
            Try { FileSystem.getFileSystemClass(scheme, hadoopConf) }.isFailure
        }

        def downloadResource(resource: String): String = {
            val uri = Utils.resolveURI(resource)
            uri.getScheme match {
                case "local" | "file" => resource
                case e if shouldDownload(e) =>
                val file = new File(targetDir, new Path(uri).getName)
                if (file.exists()) {
                    file.toURI.toString
                } else {
                    downloadFile(resource, targetDir, sparkConf, hadoopConf, secMgr)
                }
                case _ => uri.toString
            }
        }

        args.primaryResource = Option(args.primaryResource).map { downloadResource }.orNull
        args.files = Option(args.files).map { files =>
            Utils.stringToSeq(files).map(downloadResource).mkString(",")
        }.orNull
        args.pyFiles = Option(args.pyFiles).map { pyFiles =>
            Utils.stringToSeq(pyFiles).map(downloadResource).mkString(",")
        }.orNull
        args.jars = Option(args.jars).map { jars =>
            Utils.stringToSeq(jars).map(downloadResource).mkString(",")
        }.orNull
        args.archives = Option(args.archives).map { archives =>
            Utils.stringToSeq(archives).map(downloadResource).mkString(",")
        }.orNull
    }

    // If we're running a python app, set the main class to our specific python runner
    if (args.isPython && deployMode == CLIENT) {
        if (args.primaryResource == PYSPARK_SHELL) {
            args.mainClass = "org.apache.spark.api.python.PythonGatewayServer"
        } else {
            // If a python file is provided, add it to the child arguments and list of files to deploy.
            // Usage: PythonAppRunner <main python file> <extra python files> [app arguments]
            args.mainClass = "org.apache.spark.deploy.PythonRunner"
            args.childArgs = ArrayBuffer(localPrimaryResource, localPyFiles) ++ args.childArgs
        }
        if (clusterManager != YARN) {
            // The YARN backend handles python files differently, so don't merge the lists.
            args.files = mergeFileLists(args.files, args.pyFiles)
        }
    }

    if (localPyFiles != null) {
        sparkConf.set("spark.submit.pyFiles", localPyFiles)
    }

    // In YARN mode for an R app, add the SparkR package archive and the R package
    // archive containing all of the built R libraries to archives so that they can
    // be distributed with the job
    if (args.isR && clusterManager == YARN) {
        val sparkRPackagePath = RUtils.localSparkRPackagePath
        if (sparkRPackagePath.isEmpty) {
            error("SPARK_HOME does not exist for R application in YARN mode.")
        }
        val sparkRPackageFile = new File(sparkRPackagePath.get, SPARKR_PACKAGE_ARCHIVE)
        if (!sparkRPackageFile.exists()) {
            error(s"$SPARKR_PACKAGE_ARCHIVE does not exist for R application in YARN mode.")
        }
        val sparkRPackageURI = Utils.resolveURI(sparkRPackageFile.getAbsolutePath).toString

        // Distribute the SparkR package.
        // Assigns a symbol link name "sparkr" to the shipped package.
        args.archives = mergeFileLists(args.archives, sparkRPackageURI + "#sparkr")

        // Distribute the R package archive containing all the built R packages.
        if (!RUtils.rPackages.isEmpty) {
            val rPackageFile =
            RPackageUtils.zipRLibraries(new File(RUtils.rPackages.get), R_PACKAGE_ARCHIVE)
            if (!rPackageFile.exists()) {
                error("Failed to zip all the built R packages.")
            }

            val rPackageURI = Utils.resolveURI(rPackageFile.getAbsolutePath).toString
            // Assigns a symbol link name "rpkg" to the shipped package.
            args.archives = mergeFileLists(args.archives, rPackageURI + "#rpkg")
        }
    }

    // TODO: Support distributing R packages with standalone cluster
    if (args.isR && clusterManager == STANDALONE && !RUtils.rPackages.isEmpty) {
        error("Distributing R packages with standalone cluster is not supported.")
    }

    // TODO: Support distributing R packages with mesos cluster
    if (args.isR && clusterManager == MESOS && !RUtils.rPackages.isEmpty) {
        error("Distributing R packages with mesos cluster is not supported.")
    }

    // If we're running an R app, set the main class to our specific R runner
    if (args.isR && deployMode == CLIENT) {
        if (args.primaryResource == SPARKR_SHELL) {
            args.mainClass = "org.apache.spark.api.r.RBackend"
        } else {
            // If an R file is provided, add it to the child arguments and list of files to deploy.
            // Usage: RRunner <main R file> [app arguments]
            args.mainClass = "org.apache.spark.deploy.RRunner"
            args.childArgs = ArrayBuffer(localPrimaryResource) ++ args.childArgs
            args.files = mergeFileLists(args.files, args.primaryResource)
        }
    }

    if (isYarnCluster && args.isR) {
        // In yarn-cluster mode for an R app, add primary resource to files
        // that can be distributed with the job
        args.files = mergeFileLists(args.files, args.primaryResource)
    }

    // Special flag to avoid deprecation warnings at the client
    sys.props("SPARK_SUBMIT") = "true"

    // A list of rules to map each argument to system properties or command-line options in
    // each deploy mode; we iterate through these below
    val options = List[OptionAssigner](

        // All cluster managers
        OptionAssigner(args.master, ALL_CLUSTER_MGRS, ALL_DEPLOY_MODES, confKey = "spark.master"),
        OptionAssigner(args.deployMode, ALL_CLUSTER_MGRS, ALL_DEPLOY_MODES,
                       confKey = "spark.submit.deployMode"),
        OptionAssigner(args.name, ALL_CLUSTER_MGRS, ALL_DEPLOY_MODES, confKey = "spark.app.name"),
        OptionAssigner(args.ivyRepoPath, ALL_CLUSTER_MGRS, CLIENT, confKey = "spark.jars.ivy"),
        OptionAssigner(args.driverMemory, ALL_CLUSTER_MGRS, CLIENT,
                       confKey = "spark.driver.memory"),
        OptionAssigner(args.driverExtraClassPath, ALL_CLUSTER_MGRS, ALL_DEPLOY_MODES,
                       confKey = "spark.driver.extraClassPath"),
        OptionAssigner(args.driverExtraJavaOptions, ALL_CLUSTER_MGRS, ALL_DEPLOY_MODES,
                       confKey = "spark.driver.extraJavaOptions"),
        OptionAssigner(args.driverExtraLibraryPath, ALL_CLUSTER_MGRS, ALL_DEPLOY_MODES,
                       confKey = "spark.driver.extraLibraryPath"),

        // Propagate attributes for dependency resolution at the driver side
        OptionAssigner(args.packages, STANDALONE | MESOS, CLUSTER, confKey = "spark.jars.packages"),
        OptionAssigner(args.repositories, STANDALONE | MESOS, CLUSTER,
                       confKey = "spark.jars.repositories"),
        OptionAssigner(args.ivyRepoPath, STANDALONE | MESOS, CLUSTER, confKey = "spark.jars.ivy"),
        OptionAssigner(args.packagesExclusions, STANDALONE | MESOS,
                       CLUSTER, confKey = "spark.jars.excludes"),

        // Yarn only
        OptionAssigner(args.queue, YARN, ALL_DEPLOY_MODES, confKey = "spark.yarn.queue"),
        OptionAssigner(args.numExecutors, YARN, ALL_DEPLOY_MODES,
                       confKey = "spark.executor.instances"),
        OptionAssigner(args.pyFiles, YARN, ALL_DEPLOY_MODES, confKey = "spark.yarn.dist.pyFiles"),
        OptionAssigner(args.jars, YARN, ALL_DEPLOY_MODES, confKey = "spark.yarn.dist.jars"),
        OptionAssigner(args.files, YARN, ALL_DEPLOY_MODES, confKey = "spark.yarn.dist.files"),
        OptionAssigner(args.archives, YARN, ALL_DEPLOY_MODES, confKey = "spark.yarn.dist.archives"),
        OptionAssigner(args.principal, YARN, ALL_DEPLOY_MODES, confKey = "spark.yarn.principal"),
        OptionAssigner(args.keytab, YARN, ALL_DEPLOY_MODES, confKey = "spark.yarn.keytab"),

        // Other options
        OptionAssigner(args.executorCores, STANDALONE | YARN | KUBERNETES, ALL_DEPLOY_MODES,
                       confKey = "spark.executor.cores"),
        OptionAssigner(args.executorMemory, STANDALONE | MESOS | YARN | KUBERNETES, ALL_DEPLOY_MODES,
                       confKey = "spark.executor.memory"),
        OptionAssigner(args.totalExecutorCores, STANDALONE | MESOS | KUBERNETES, ALL_DEPLOY_MODES,
                       confKey = "spark.cores.max"),
        OptionAssigner(args.files, LOCAL | STANDALONE | MESOS | KUBERNETES, ALL_DEPLOY_MODES,
                       confKey = "spark.files"),
        OptionAssigner(args.jars, LOCAL, CLIENT, confKey = "spark.jars"),
        OptionAssigner(args.jars, STANDALONE | MESOS | KUBERNETES, ALL_DEPLOY_MODES,
                       confKey = "spark.jars"),
        OptionAssigner(args.driverMemory, STANDALONE | MESOS | YARN | KUBERNETES, CLUSTER,
                       confKey = "spark.driver.memory"),
        OptionAssigner(args.driverCores, STANDALONE | MESOS | YARN | KUBERNETES, CLUSTER,
                       confKey = "spark.driver.cores"),
        OptionAssigner(args.supervise.toString, STANDALONE | MESOS, CLUSTER,
                       confKey = "spark.driver.supervise"),
        OptionAssigner(args.ivyRepoPath, STANDALONE, CLUSTER, confKey = "spark.jars.ivy"),

        // An internal option used only for spark-shell to add user jars to repl's classloader,
        // previously it uses "spark.jars" or "spark.yarn.dist.jars" which now may be pointed to
        // remote jars, so adding a new option to only specify local jars for spark-shell internally.
        OptionAssigner(localJars, ALL_CLUSTER_MGRS, CLIENT, confKey = "spark.repl.local.jars")
    )

    // In client mode, launch the application main class directly
    // In addition, add the main application jar and any added jars (if any) to the classpath
    if (deployMode == CLIENT) {
        // 如果是客户端模式，childMainClass就是用户Job运行的主类，即Driver在运行提交程序的JVM中
        childMainClass = args.mainClass
        if (localPrimaryResource != null && isUserJar(localPrimaryResource)) {
            childClasspath += localPrimaryResource
        }
        if (localJars != null) { childClasspath ++= localJars.split(",") }
    }
    // Add the main application jar and any added jars to classpath in case YARN client
    // requires these jars.
    // This assumes both primaryResource and user jars are local jars, or already downloaded
    // to local by configuring "spark.yarn.dist.forceDownloadSchemes", otherwise it will not be
    // added to the classpath of YARN client.
    if (isYarnCluster) {
        if (isUserJar(args.primaryResource)) {
            childClasspath += args.primaryResource
        }
        if (args.jars != null) { childClasspath ++= args.jars.split(",") }
    }

    if (deployMode == CLIENT) {
        if (args.childArgs != null) { childArgs ++= args.childArgs }
    }

    // Map all arguments to command-line options or system properties for our chosen mode
    for (opt <- options) {
        if (opt.value != null &&
            (deployMode & opt.deployMode) != 0 &&
            (clusterManager & opt.clusterManager) != 0) {
            if (opt.clOption != null) { childArgs += (opt.clOption, opt.value) }
            if (opt.confKey != null) { sparkConf.set(opt.confKey, opt.value) }
        }
    }

    // In case of shells, spark.ui.showConsoleProgress can be true by default or by user.
    if (isShell(args.primaryResource) && !sparkConf.contains(UI_SHOW_CONSOLE_PROGRESS)) {
        sparkConf.set(UI_SHOW_CONSOLE_PROGRESS, true)
    }

    // Add the application jar automatically so the user doesn't have to call sc.addJar
    // For YARN cluster mode, the jar is already distributed on each node as "app.jar"
    // For python and R files, the primary resource is already distributed as a regular file
    if (!isYarnCluster && !args.isPython && !args.isR) {
        var jars = sparkConf.getOption("spark.jars").map(x => x.split(",").toSeq).getOrElse(Seq.empty)
        if (isUserJar(args.primaryResource)) {
            jars = jars ++ Seq(args.primaryResource)
        }
        sparkConf.set("spark.jars", jars.mkString(","))
    }

    // In standalone cluster mode, use the REST client to submit the application (Spark 1.3+).
    // 在standalone模式下，Spark1.3 之后使用Rest 客户端提交Application
    // All Spark parameters are expected to be passed to the client through system properties.
    if (args.isStandaloneCluster) {
        if (args.useRest) {
            childMainClass = REST_CLUSTER_SUBMIT_CLASS
            childArgs += (args.primaryResource, args.mainClass)
        } else {
            // In legacy standalone cluster mode, use Client as a wrapper around the user class
            childMainClass = STANDALONE_CLUSTER_SUBMIT_CLASS
            if (args.supervise) { childArgs += "--supervise" }
            Option(args.driverMemory).foreach { m => childArgs += ("--memory", m) }
            Option(args.driverCores).foreach { c => childArgs += ("--cores", c) }
            childArgs += "launch"
            childArgs += (args.master, args.primaryResource, args.mainClass)
        }
        if (args.childArgs != null) {
            childArgs ++= args.childArgs
        }
    }

    // Let YARN know it's a pyspark app, so it distributes needed libraries.
    if (clusterManager == YARN) {
        if (args.isPython) {
            sparkConf.set("spark.yarn.isPython", "true")
        }
    }

    if (clusterManager == MESOS && UserGroupInformation.isSecurityEnabled) {
        setRMPrincipal(sparkConf)
    }

    // In yarn-cluster mode, use yarn.Client as a wrapper around the user class
    // 在yarn-cluster模式下，使用org.apache.spark.deploy.yarn.YarnClusterApplication提交Application
    if (isYarnCluster) {
        childMainClass = YARN_CLUSTER_SUBMIT_CLASS
        if (args.isPython) {
            childArgs += ("--primary-py-file", args.primaryResource)
            childArgs += ("--class", "org.apache.spark.deploy.PythonRunner")
        } else if (args.isR) {
            val mainFile = new Path(args.primaryResource).getName
            childArgs += ("--primary-r-file", mainFile)
            childArgs += ("--class", "org.apache.spark.deploy.RRunner")
        } else {
            if (args.primaryResource != SparkLauncher.NO_RESOURCE) {
                childArgs += ("--jar", args.primaryResource)
            }
            childArgs += ("--class", args.mainClass)
        }
        if (args.childArgs != null) {
            args.childArgs.foreach { arg => childArgs += ("--arg", arg) }
        }
    }

    if (isMesosCluster) {
        assert(args.useRest, "Mesos cluster mode is only supported through the REST submission API")
        childMainClass = REST_CLUSTER_SUBMIT_CLASS
        if (args.isPython) {
            // Second argument is main class
            childArgs += (args.primaryResource, "")
            if (args.pyFiles != null) {
                sparkConf.set("spark.submit.pyFiles", args.pyFiles)
            }
        } else if (args.isR) {
            // Second argument is main class
            childArgs += (args.primaryResource, "")
        } else {
            childArgs += (args.primaryResource, args.mainClass)
        }
        if (args.childArgs != null) {
            childArgs ++= args.childArgs
        }
    }

    if (isKubernetesCluster) {
        childMainClass = KUBERNETES_CLUSTER_SUBMIT_CLASS
        if (args.primaryResource != SparkLauncher.NO_RESOURCE) {
            if (args.isPython) {
                childArgs ++= Array("--primary-py-file", args.primaryResource)
                childArgs ++= Array("--main-class", "org.apache.spark.deploy.PythonRunner")
                if (args.pyFiles != null) {
                    childArgs ++= Array("--other-py-files", args.pyFiles)
                }
            } else if (args.isR) {
                childArgs ++= Array("--primary-r-file", args.primaryResource)
                childArgs ++= Array("--main-class", "org.apache.spark.deploy.RRunner")
            }
            else {
                childArgs ++= Array("--primary-java-resource", args.primaryResource)
                childArgs ++= Array("--main-class", args.mainClass)
            }
        } else {
            childArgs ++= Array("--main-class", args.mainClass)
        }
        if (args.childArgs != null) {
            args.childArgs.foreach { arg =>
                childArgs += ("--arg", arg)
            }
        }
    }

    // Load any properties specified through --conf and the default properties file
    for ((k, v) <- args.sparkProperties) {
        sparkConf.setIfMissing(k, v)
    }

    // Ignore invalid spark.driver.host in cluster modes.
    if (deployMode == CLUSTER) {
        sparkConf.remove("spark.driver.host")
    }

    // Resolve paths in certain spark properties
    val pathConfigs = Seq(
        "spark.jars",
        "spark.files",
        "spark.yarn.dist.files",
        "spark.yarn.dist.archives",
        "spark.yarn.dist.jars")
    pathConfigs.foreach { config =>
        // Replace old URIs with resolved URIs, if they exist
        sparkConf.getOption(config).foreach { oldValue =>
            sparkConf.set(config, Utils.resolveURIs(oldValue))
        }
    }

    // Resolve and format python file paths properly before adding them to the PYTHONPATH.
    // The resolving part is redundant in the case of --py-files, but necessary if the user
    // explicitly sets `spark.submit.pyFiles` in his/her default properties file.
    sparkConf.getOption("spark.submit.pyFiles").foreach { pyFiles =>
        val resolvedPyFiles = Utils.resolveURIs(pyFiles)
        val formattedPyFiles = if (!isYarnCluster && !isMesosCluster) {
            PythonRunner.formatPaths(resolvedPyFiles).mkString(",")
        } else {
            // Ignoring formatting python path in yarn and mesos cluster mode, these two modes
            // support dealing with remote python files, they could distribute and add python files
            // locally.
            resolvedPyFiles
        }
        sparkConf.set("spark.submit.pyFiles", formattedPyFiles)
    }

    (childArgs, childClasspath, sparkConf, childMainClass)
}
```

## 3. 总结

SparkSubmit提交Application是调用：

```java
// childMainClass决定Application调用哪个类提交
mainClass = Utils.classForName(childMainClass)
```

（1）当--deploy-mode:cluster且--master: yarn时，会调用YarnClusterApplication进行提交，YarnClusterApplication这是org.apache.spark.deploy.yarn.Client中的一个内部类，在YarnClusterApplication中new了一个Client对象，并调用了run方法。

（2）当--deploy-mode:client时，调用application-jar.jar自身main函数，执行的是JavaMainApplication

