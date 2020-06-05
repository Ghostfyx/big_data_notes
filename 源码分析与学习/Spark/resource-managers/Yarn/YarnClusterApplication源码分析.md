# YarnClusterApplication 源码分析

## 1. 简介

SparkSubmit类中：

`YARN_CLUSTER_SUBMIT_CLASS = "org.apache.spark.deploy.yarn.YarnClusterApplication"`

YARN集群模式下调用YarnClusterApplication类与YARN集群交互，提交Application。

YarnClusterApplication这是org.apache.spark.deploy.yarn.Client中的一个内部类，在YarnClusterApplication中new了一个Client对象，并调用了run方法

## 2. 源码

```java
private[spark] class YarnClusterApplication extends SparkApplication {

  override def start(args: Array[String], conf: SparkConf): Unit = {
    // SparkSubmit would use yarn cache to distribute files & jars in yarn mode,
    // so remove them from sparkConf here for yarn mode.
    conf.remove("spark.jars")
    conf.remove("spark.files")
    
    // 实例化Client类，并调用run方法
    new Client(new ClientArguments(args), conf).run()
  }
    
}
```

