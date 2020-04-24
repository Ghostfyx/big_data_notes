# Hadoop: Writing YARN Applications

 ## 1. 概述

本节描述了如何使用高阶API开发YARN应用。

## 2. 开发概念与流程

一般概念是，应用程序提交客户端将应用程序提交到YARN ResourceManager(RM)。可以通过创建YarnClient对象来完成，启动YarnClient之后，客户端可以设置应用程序上下文，准备包含ApplicationMaster(AM)的应用程序的第一个容器，然后提交应用程序。需要提供的信息：例如关于运行应用所需的本地文件/Jar的详细信息，需要执行的实际命令(带有必要的命令行参数)，任何OS环境设置(可选)等等，实际上，需要描述为ApplicationMaster启动的Unix进程。

然后，YARN ResourceManager将在分配的容器上启动ApplicationMaster(如指定)。ApplicationMaster与YARN集群通信，并处理应用程序执行。它以异步方式执行操作。在应用程序启动期间，ApplicationMaster的主要任务是：

a）与ResourceManager进行通信以协商和分配将来的容器资源；

b）容器分配后，与YARN NodeManager(NM)进行通信启动NM上的应用程序容器。

任务a）可以通过AMRMClientAsync对象异步执行，使用事件处理程序的AMRMClientAsync.CallbackHandler类型中指定的事件处理方法。需要将事件处理程序显式设置为客户端。任务b）可以通过启动可运行对象来执行，然后在分配了容器后启动容器。作为启动此容器的一部分，AM必须指定具有启动信息(例如命令行规范，环境等)的ContainerLaunchContext。

在应用程序执行期间，ApplicationMaster通过NMClientAsync对象与NodeManager通信。所有容器事件均由与NMClientAsync关联的NMClientAsync.CallbackHandler处理，典型的回调处理程序处理客户端的启动，停止，状态更新和错误。ApplicationMaster还通过处理AMRMClientAsync.CallbackHandler的getProgress()方法向ResourceManager报告Job执行进度。

除了异步客户端，某些工作流(AMRMClient与NMClient)还有同步执行的版本。 但是推荐使用异步客户端是因为更简单的用法，本文将主要介绍异步客户端。 有关同步客户端的更多信息，请参考AMRMClient和NMClient。

## 3. 接口

以下是重要的接口：

- **Client**<-->**ResourceManager**  通过使用YarnClient对象
- **ApplicationMaster**<-->**ResourceManager**   通过使用AMRMClientAsync对象，AMRMClientAsync.CallbackHandler异步处理事件
- **ApplicationMaster**<-->**NodeManager**  启动容器。 通过使用NMClientAsync对象与NodeManager进行通信，并通过NMClientAsync.CallbackHandler处理容器事件

注意：

- YARN应用程序的三个主要协议（ApplicationClientProtocol，ApplicationMasterProtocol和ContainerManagementProtocol）仍然保留。 3个客户端包装了这3个协议，以为YARN应用程序提供更简单的编程模型。
- 在极少数情况下，程序员可能希望直接使用这三种协议来实现应用程序。 但是，请注意，对于一般用例，不再鼓励此类行为。

## 4. 写一个简单的YARN应用程序

### 4.1 开发一个简单的客户端

- 第一步初始化并启动YarnClient

	```java
	  YarnClient yarnClient = YarnClient.createYarnClient();
	  yarnClient.init(conf);
	  yarnClient.start();
	```

- 设置客户端后，客户端需要创建一个应用程序，并获取其应用程序ID

	```java
	  YarnClientApplication app = yarnClient.createApplication();
	  GetNewApplicationResponse appResponse = app.getNewApplicationResponse();
	```

- YarnClientApplication对新应用程序的响应还包含有关群集的信息，例如群集的最小/最大资源能力。这是必需的，以便确保可以正确设置将在其中启动ApplicationMaster的容器的规格。 有关更多详细信息，请参考GetNewApplicationResponse。

- 客户端的主要关键在于设置ApplicationSubmissionContext，它定义RM启动AM所需的所有信息。 客户需要在上下文中设置以下内容：

	1）应用信息，例如：id，name

	2）队列，优先级消息：提交申请的队列，为申请分配的优先级

	3）用户：提交应用的用户

	4）ContainerLaunchContext：定义将在其中启动和运行AM的容器的信息。 如前所述，ContainerLaunchContext定义了运行应用程序所需的所有必需信息，例如本地资源：二进制文件，jar，文件等，环境设置(CLASSPATH等)，要执行的命令和安全令牌(RECT)。

```java
// set the application submission context
ApplicationSubmissionContext appContext = app.getApplicationSubmissionContext();
ApplicationId appId = appContext.getApplicationId();

appContext.setKeepContainersAcrossApplicationAttempts(keepContainers);
appContext.setApplicationName(appName);

// set local resources for the application master
// local files or archives as needed
// In this scenario, the jar file for the application master is part of the local resources
Map<String, LocalResource> localResources = new HashMap<String, LocalResource>();

LOG.info("Copy App Master jar from local filesystem and add to local environment");
// Copy the application master jar to the filesystem
// Create a local resource to point to the destination jar path
FileSystem fs = FileSystem.get(conf);
addToLocalResources(fs, appMasterJar, appMasterJarPath, appId.toString(),
    localResources, null);

// Set the log4j properties if needed
if (!log4jPropFile.isEmpty()) {
  addToLocalResources(fs, log4jPropFile, log4jPath, appId.toString(),
      localResources, null);
}

// The shell script has to be made available on the final container(s)
// where it will be executed.
// To do this, we need to first copy into the filesystem that is visible
// to the yarn framework.
// We do not need to set this as a local resource for the application
// master as the application master does not need it.
String hdfsShellScriptLocation = "";
long hdfsShellScriptLen = 0;
long hdfsShellScriptTimestamp = 0;
if (!shellScriptPath.isEmpty()) {
  Path shellSrc = new Path(shellScriptPath);
  String shellPathSuffix =
      appName + "/" + appId.toString() + "/" + SCRIPT_PATH;
  Path shellDst =
      new Path(fs.getHomeDirectory(), shellPathSuffix);
  fs.copyFromLocalFile(false, true, shellSrc, shellDst);
  hdfsShellScriptLocation = shellDst.toUri().toString();
  FileStatus shellFileStatus = fs.getFileStatus(shellDst);
  hdfsShellScriptLen = shellFileStatus.getLen();
  hdfsShellScriptTimestamp = shellFileStatus.getModificationTime();
}

if (!shellCommand.isEmpty()) {
  addToLocalResources(fs, null, shellCommandPath, appId.toString(),
      localResources, shellCommand);
}

if (shellArgs.length > 0) {
  addToLocalResources(fs, null, shellArgsPath, appId.toString(),
      localResources, StringUtils.join(shellArgs, " "));
}

// Set the env variables to be setup in the env where the application master will be run
LOG.info("Set the environment for the application master");
Map<String, String> env = new HashMap<String, String>();

// put location of shell script into env
// using the env info, the application master will create the correct local resource for the
// eventual containers that will be launched to execute the shell scripts
env.put(DSConstants.DISTRIBUTEDSHELLSCRIPTLOCATION, hdfsShellScriptLocation);
env.put(DSConstants.DISTRIBUTEDSHELLSCRIPTTIMESTAMP, Long.toString(hdfsShellScriptTimestamp));
env.put(DSConstants.DISTRIBUTEDSHELLSCRIPTLEN, Long.toString(hdfsShellScriptLen));

// Add AppMaster.jar location to classpath
// At some point we should not be required to add
// the hadoop specific classpaths to the env.
// It should be provided out of the box.
// For now setting all required classpaths including
// the classpath to "." for the application jar
StringBuilder classPathEnv = new StringBuilder(Environment.CLASSPATH.$$())
  .append(ApplicationConstants.CLASS_PATH_SEPARATOR).append("./*");
for (String c : conf.getStrings(
    YarnConfiguration.YARN_APPLICATION_CLASSPATH,
    YarnConfiguration.DEFAULT_YARN_CROSS_PLATFORM_APPLICATION_CLASSPATH)) {
  classPathEnv.append(ApplicationConstants.CLASS_PATH_SEPARATOR);
  classPathEnv.append(c.trim());
}
classPathEnv.append(ApplicationConstants.CLASS_PATH_SEPARATOR).append(
  "./log4j.properties");

// Set the necessary command to execute the application master
Vector<CharSequence> vargs = new Vector<CharSequence>(30);

// Set java executable command
LOG.info("Setting up app master command");
vargs.add(Environment.JAVA_HOME.$$() + "/bin/java");
// Set Xmx based on am memory size
vargs.add("-Xmx" + amMemory + "m");
// Set class name
vargs.add(appMasterMainClass);
// Set params for Application Master
vargs.add("--container_memory " + String.valueOf(containerMemory));
vargs.add("--container_vcores " + String.valueOf(containerVirtualCores));
vargs.add("--num_containers " + String.valueOf(numContainers));
vargs.add("--priority " + String.valueOf(shellCmdPriority));

for (Map.Entry<String, String> entry : shellEnv.entrySet()) {
  vargs.add("--shell_env " + entry.getKey() + "=" + entry.getValue());
}
if (debugFlag) {
  vargs.add("--debug");
}

vargs.add("1>" + ApplicationConstants.LOG_DIR_EXPANSION_VAR + "/AppMaster.stdout");
vargs.add("2>" + ApplicationConstants.LOG_DIR_EXPANSION_VAR + "/AppMaster.stderr");

// Get final command
StringBuilder command = new StringBuilder();
for (CharSequence str : vargs) {
  command.append(str).append(" ");
}

LOG.info("Completed setting up app master command " + command.toString());
List<String> commands = new ArrayList<String>();
commands.add(command.toString());

// Set up the container launch context for the application master
ContainerLaunchContext amContainer = ContainerLaunchContext.newInstance(
  localResources, env, commands, null, null, null);

// Set up resource type requirements
// For now, both memory and vcores are supported, so we set memory and
// vcores requirements
Resource capability = Resource.newInstance(amMemory, amVCores);
appContext.setResource(capability);

// Service data is a binary blob that can be passed to the application
// Not needed in this scenario
// amContainer.setServiceData(serviceData);

// Setup security tokens
if (UserGroupInformation.isSecurityEnabled()) {
  // Note: Credentials class is marked as LimitedPrivate for HDFS and MapReduce
  Credentials credentials = new Credentials();
  String tokenRenewer = conf.get(YarnConfiguration.RM_PRINCIPAL);
  if (tokenRenewer == null | | tokenRenewer.length() == 0) {
    throw new IOException(
      "Can't get Master Kerberos principal for the RM to use as renewer");
  }

  // For now, only getting tokens for the default file-system.
  final Token<?> tokens[] =
      fs.addDelegationTokens(tokenRenewer, credentials);
  if (tokens != null) {
    for (Token<?> token : tokens) {
      LOG.info("Got dt for " + fs.getUri() + "; " + token);
    }
  }
  DataOutputBuffer dob = new DataOutputBuffer();
  credentials.writeTokenStorageToStream(dob);
  ByteBuffer fsTokens = ByteBuffer.wrap(dob.getData(), 0, dob.getLength());
  amContainer.setTokens(fsTokens);
}

appContext.setAMContainerSpec(amContainer);
```

- 设置过程完成后，客户端即可准备以指定的优先级和队列提交应用程序。

	```java
	// Set the priority for the application master
	Priority pri = Priority.newInstance(amPriority);
	appContext.setPriority(pri);
	
	// Set the queue to which this application is to be submitted in the RM
	appContext.setQueue(amQueue);
	
	// Submit the application to the applications manager
	// SubmitApplicationResponse submitResp = applicationsManager.submitApplication(appRequest);
	
	yarnClient.submitApplication(appContext);
	```

- 至此，RM将在后台接受该应用程序，并将分配具有所需规范的容器，然后最终在所分配的容器上设置并启动AM的过程。

- 客户可以通过多种方式跟踪实际任务的进度:

	- 可以与RM通信，并通过YarnClient的getApplicationReport()方法请求应用程序报告。

		```java
		// Get application report for the appId we are interested in
		ApplicationReport report = yarnClient.getApplicationReport(appId);
		```

		​	从RM收到的ApplicationReport包括以下内容：

				- *General application information*：应用程序ID，提交应用程序的队列，提交应用程序的用户以及应用程序的开始时间
				- *ApplicationMaster details*：运行AM的主机，正在监听来自客户端的请求的rpc端口以及客户端需要与AM通信的令牌
				- *Application tracking information*：如果应用程序支持某种形式的进度跟踪，则可以设置一个跟踪URL，该URL可通过ApplicationReport的getTrackingUrl方法获得，客户端可以查看该URL来监视进度
				- *Application status*：ResourceManager看到的应用程序状态可通过ApplicationReport.getYarnApplicationState获得

	- 如果ApplicationMaster支持，则客户端可以通过从应用程序报告中获取的host:rpcport信息直接向AM本身查询进度更新。 如果可用，它也可以使用从报告中获得的跟踪URL。

- 在某些情况下，如果应用程序花费的时间太长或由于其他因素，客户端可能希望终止该应用程序。 YarnClient支持killApplication调用，该调用允许客户端通过ResourceManager向AM发送终止信号。 如果这样设计，ApplicationMaster还可以通过客户端可能利用的rpc层支持中止调用

	```java
	  yarnClient.killApplication(appId);
	```

### 4.2 开发一个ApplicationMaster

- AM是工作的实际所有者，它将由RM启动，并且将通过客户端向客户提供有关已负责监督和完成的工作的所有必要信息和资源
- 考虑到多租户性质，由于AM可能在与其他容器共享物理主机的容器中启动的，除其他问题外，它无法对可以监听的预配置端口等任何假设
- 当AM启动，通过环境可以使用几个参数：其中包括AM容器的ContainerId，应用程序提交时间以及有关运行ApplicationMaster的NM主机的详细信息。 有关参数名称，请参阅ApplicationConstants
- 与RM的所有交互都需要一个ApplicationAttemptId(如果出现故障，每个应用会多次尝试)。可以从AM的容器ID中获取ApplicationAttemptId。 有一些帮助程序API可以将从环境中获取的值转换为对象。

```java
Map<String, String> envs = System.getenv();
String containerIdString =
    envs.get(ApplicationConstants.AM_CONTAINER_ID_ENV);
if (containerIdString == null) {
  // container id should always be set in the env by the framework
  throw new IllegalArgumentException(
      "ContainerId not set in the environment");
}
ContainerId containerId = ConverterUtils.toContainerId(containerIdString);
ApplicationAttemptId appAttemptID = containerId.getApplicationAttemptId();
```

- 在AM完全初始化其自身之后，可以启动两个客户端：一个到ResourceManager，另一个到NodeManagers。 使用自定义的事件处理程序来设置它们，将在本文后面详细讨论这些事件处理程序。

```java
  AMRMClientAsync.CallbackHandler allocListener = new RMCallbackHandler();
  amRMClient = AMRMClientAsync.createAMRMClientAsync(1000, allocListener);
  amRMClient.init(conf);
  amRMClient.start();

  containerListener = createNMCallbackHandler();
  nmClientAsync = new NMClientAsyncImpl(containerListener);
  nmClientAsync.init(conf);
  nmClientAsync.start();
```

- AM必须向RM发出心跳，以通知其AM仍在运行并且仍在运行。 RM的超时到期间隔由可通过YarnConfiguration.RM_AM_EXPIRY_INTERVAL_MS访问的配置设置定义，默认设置由YarnConfiguration.DEFAULT_RM_AM_EXPIRY_INTERVAL_MS定义。 ApplicationMaster需要向ResourceManager注册以启动心跳。

```java
// Register self with ResourceManager
// This will start heartbeating to the RM
appMasterHostname = NetUtils.getHostname();
RegisterApplicationMasterResponse response = amRMClient
    .registerApplicationMaster(appMasterHostname, appMasterRpcPort,
        appMasterTrackingUrl);
```

- 在注册响应中，包括最大资源容量。 可能要使用它来检查应用程序的请求。

```java
// Dump out information about cluster capability as seen by the
// resource manager
int maxMem = response.getMaximumResourceCapability().getMemory();
LOG.info("Max mem capability of resources in this cluster " + maxMem);

int maxVCores = response.getMaximumResourceCapability().getVirtualCores();
LOG.info("Max vcores capability of resources in this cluster " + maxVCores);

// A resource ask cannot exceed the max.
if (containerMemory > maxMem) {
  LOG.info("Container memory specified above max threshold of cluster."
      + " Using max value." + ", specified=" + containerMemory + ", max="
      + maxMem);
  containerMemory = maxMem;
}

if (containerVirtualCores > maxVCores) {
  LOG.info("Container virtual cores specified above max threshold of  cluster."
    + " Using max value." + ", specified=" + containerVirtualCores + ", max="
    + maxVCores);
  containerVirtualCores = maxVCores;
}
List<Container> previousAMRunningContainers =
    response.getContainersFromPreviousAttempts();
LOG.info("Received " + previousAMRunningContainers.size()
        + " previous AM's running containers on AM registration.");
```

- 根据任务需求，AM可以请求一组容器来运行其任务。 现在，我们可以计算需要多少个容器，并请求那么多个容器。

```java
List<Container> previousAMRunningContainers =
    response.getContainersFromPreviousAttempts();
LOG.info("Received " + previousAMRunningContainers.size()
    + " previous AM's running containers on AM registration.");

int numTotalContainersToRequest =
    numTotalContainers - previousAMRunningContainers.size();
// Setup ask for containers from RM
// Send request for containers to RM
// Until we get our fully allocated quota, we keep on polling RM for
// containers
// Keep looping until all the containers are launched and shell script
// executed on them ( regardless of success/failure).
for (int i = 0; i < numTotalContainersToRequest; ++i) {
  ContainerRequest containerAsk = setupContainerAskForRM();
  amRMClient.addContainerRequest(containerAsk);
}
```

- 在setupContainerAskForRM()中，需要设置以下两件事：

	- 资源功能：目前，YARN支持基于内存的资源要求，因此请求应定义所需的内存量。该值以MB为单位定义，并且必须小于群集的最大容量，并且必须是最小容量的精确倍数。 内存资源对应于任务容器上施加的物理内存限制。 如代码所示，它还将支持用于计算的资源(vCore)
	- 优先级：当请求容器集时，AM可以为每个集合定义不同的优先级。 例如，Map-Reduce AM可以为Map任务所需的容器分配较高的优先级，而为Reduce任务的容器分配较低的优先级

	```java
	private ContainerRequest setupContainerAskForRM() {
	  // setup requirements for hosts
	  // using * as any host will do for the distributed shell app
	  // set the priority for the request
	  Priority pri = Priority.newInstance(requestPriority);
	
	  // Set up resource type requirements
	  // For now, memory and CPU are supported so we set memory and cpu requirements
	  Resource capability = Resource.newInstance(containerMemory,
	    containerVirtualCores);
	
	  ContainerRequest request = new ContainerRequest(capability, null, null,
	      pri);
	  LOG.info("Requested container ask: " + request.toString());
	  return request;
	}
	```

- 在ApplicationMaster发送了容器分配请求之后，会通过AMRMClientAsync客户端的事件处理程序异步启动容器。 处理程序应实现AMRMClientAsync.CallbackHandler接口。

	- 分配了容器后，处理程序将建立一个线程，该线程运行代码以启动容器。 在这里，使用名称LaunchContainerRunnable进行演示。 将在本文的以下部分中讨论LaunchContainerRunnable类

		```java
		@Override
		public void onContainersAllocated(List<Container> allocatedContainers) {
		  LOG.info("Got response from RM for container ask, allocatedCnt="
		      + allocatedContainers.size());
		  numAllocatedContainers.addAndGet(allocatedContainers.size());
		  for (Container allocatedContainer : allocatedContainers) {
		    LaunchContainerRunnable runnableLaunchContainer =
		        new LaunchContainerRunnable(allocatedContainer, containerListener);
		    Thread launchThread = new Thread(runnableLaunchContainer);
		
		    // launch and start the container on a separate thread to keep
		    // the main thread unblocked
		    // as all containers may not be allocated at one go.
		    launchThreads.add(launchThread);
		    launchThread.start();
		  }
		}
		```

	- 心跳时，事件处理程序将报告应用程序的进度

		```java
		@Override
		public float getProgress() {
		  // set progress to deliver to RM on next heartbeat
		  float progress = (float) numCompletedContainers.get()
		      / numTotalContainers;
		  return progress;
		}
		```

- 容器启动线程实际上在NM上启动容器。 在将容器分配给AM之后，它需要遵循与客户端为要在分配的Container上运行的最终任务设置ContainerLaunchContext时遵循的类似过程。定义ContainerLaunchContext后，AM可以通过NMClientAsync启动它

	```java
	// Set the necessary command to execute on the allocated container
	Vector<CharSequence> vargs = new Vector<CharSequence>(5);
	
	// Set executable command
	vargs.add(shellCommand);
	// Set shell script path
	if (!scriptPath.isEmpty()) {
	  vargs.add(Shell.WINDOWS ? ExecBatScripStringtPath
	    : ExecShellStringPath);
	}
	
	// Set args for the shell command if any
	vargs.add(shellArgs);
	// Add log redirect params
	vargs.add("1>" + ApplicationConstants.LOG_DIR_EXPANSION_VAR + "/stdout");
	vargs.add("2>" + ApplicationConstants.LOG_DIR_EXPANSION_VAR + "/stderr");
	
	// Get final command
	StringBuilder command = new StringBuilder();
	for (CharSequence str : vargs) {
	  command.append(str).append(" ");
	}
	
	List<String> commands = new ArrayList<String>();
	commands.add(command.toString());
	
	// Set up ContainerLaunchContext, setting local resource, environment,
	// command and token for constructor.
	
	// Note for tokens: Set up tokens for the container too. Today, for normal
	// shell commands, the container in distribute-shell doesn't need any
	// tokens. We are populating them mainly for NodeManagers to be able to
	// download anyfiles in the distributed file-system. The tokens are
	// otherwise also useful in cases, for e.g., when one is running a
	// "hadoop dfs" command inside the distributed shell.
	ContainerLaunchContext ctx = ContainerLaunchContext.newInstance(
	  localResources, shellEnv, commands, null, allTokens.duplicate(), null);
	containerListener.addContainer(container.getId(), container);
	nmClientAsync.startContainerAsync(container, ctx);
	```

- NMClientAsync对象及其事件处理程序一起处理容器事件。 包括容器的启动，停止，状态更新以及发生错误

- ApplicationMaster确定工作完成后，需要通过AM-RM客户端注销自身，然后停止该客户端

	```java
	try {
	  amRMClient.unregisterApplicationMaster(appStatus, appMessage, null);
	} catch (YarnException ex) {
	  LOG.error("Failed to unregister application", ex);
	} catch (IOException e) {
	  LOG.error("Failed to unregister application", e);
	}
	
	amRMClient.stop();
	```

## 5. FQA

### 5.1 如何将应用程序的jar分发到YARN群集中需要它的所有节点？

可以使用LocalResource将资源添加到应用程序请求中。 这将导致YARN将资源分发到ApplicationMaster节点。 如果资源是tgz，zip或jar可以让YARN解压缩。 然后，要做的就是将解压缩的文件夹添加到类路径中。 例如，在创建应用程序请求时：

```java
File packageFile = new File(packagePath);
URL packageUrl = ConverterUtils.getYarnUrlFromPath(
    FileContext.getFileContext().makeQualified(new Path(packagePath)));

packageResource.setResource(packageUrl);
packageResource.setSize(packageFile.length());
packageResource.setTimestamp(packageFile.lastModified());
packageResource.setType(LocalResourceType.ARCHIVE);
packageResource.setVisibility(LocalResourceVisibility.APPLICATION);

resource.setMemory(memory);
z.setResource(resource);
containerCtx.setCommands(ImmutableList.of(
    "java -cp './package/*' some.class.to.Run "
    + "1>" + ApplicationConstants.LOG_DIR_EXPANSION_VAR + "/stdout "
    + "2>" + ApplicationConstants.LOG_DIR_EXPANSION_VAR + "/stderr"));
containerCtx.setLocalResources(
    Collections.singletonMap("package", packageResource));
appCtx.setApplicationId(appId);
appCtx.setUser(user.getShortUserName);
appCtx.setAMContainerSpec(containerCtx);
yarnClient.submitApplication(appCtx);
```

setLocalResources命令采用名称到资源的映射。 该名称将成为应用程序cwd中的符号链接，因此只需使用./package/*即可引用其中的工件。

注意：Java的classpath参数非常敏感。 确保语法完全正确。

将包裹分发到AM后，只要AM启动新的容器(假设希望将资源发送到容器)，就需要遵循相同的过程。 此代码是相同的。 只需要确保为AM提供了包路径(HDFS或本地)，以便它可以将资源URL与容器ctx一起发送。

### 5.2 How do I get the ApplicationMaster’s `ApplicationAttemptId`?

ApplicationAttemptId将通过环境传递到AM，并且可以通过ConverterUtils帮助函数将来自环境的值转换为ApplicationAttemptId对象。

### 5.3 Why my container is killed by the NodeManager?

这可能是由于高内存使用量超出了您请求的容器内存大小。 有许多原因可以导致此。 首先，查看NodeManager杀死容器时转储的进程树。 两个配置是关键：物理内存和虚拟内存。 如果超出了物理内存限制，则应用程序将使用过多的物理内存。 如果正在运行Java应用程序，则可以使用-hprof查看堆中正在占用空间的内容。 如果超出了虚拟内存，则可能需要增加群集范围的配置变量yarn.nodemanager.vmem-pmem-ratio的值(该值设置虚拟内存是物理内存的倍数)。

### 5.4 How do I include native libraries?

启动容器时在命令行上设置-Djava.library.path可能导致Hadoop使用的本地库无法正确加载，并可能导致错误。 改用LD_LIBRARY_PATH更干净。