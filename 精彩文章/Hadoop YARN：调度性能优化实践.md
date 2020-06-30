# Hadoop YARN：调度性能优化实践

## 1. 整体架构

### 1.1 YARN架构

YARN负责作业资源调度，在集群中找到满足业务的资源，帮助作业启动任务，管理作业的生命周期。

### 1.2 资源抽象

YARN在cpu，memory这两个资源维度对集群资源做了抽象。

```java
class Resource{
  int cpu;   //cpu核心个数
  int memory-mb; //内存的MB数
}
```

作业向YARN申请资源的请求是：List[ResourceRequest]

```java
class ResourceRequest{
  int numContainers; //需要的container个数
  Resource capability;//每个container的资源
}
```

YARN对作业响应是：List[Container]

```java
class Container{
  ContainerId containerId; //YARN全局唯一的container标示
  Resource capability;  //该container的资源信息
  String nodeHttpAddress; //该container可以启动的NodeManager的hostname
}
```

### 1.3 YARN调度架构

![](img/Yarn_scheduler.jpg)

