# 第一章 MapReduce目录结构

## 1. MapReduce目录结构

```
hadoop-mapreduce-project
|_____bin mapreduce shell命令行模块
|_____conf mapreduce配置模块
|_____dev-support 
|_____hadoop-mapreduce-client mapreduce核心模块，重点阅读
|_____hadoop-mapreduce-example
|_____lib
|_____shellprofile.d
```

## 2. hadoop-mapreduce-client 目录结构

```xml
hadoop-mapreduce-client
|_____hadoop-mapreduce-client-core
|_____hadoop-mapreduce-client-common
|_____hadoop-mapreduce-client-shuffle
|_____hadoop-mapreduce-client-app
|_____hadoop-mapreduce-client-jobclient
|_____hadoop-mapreduce-client-hs
|_____hadoop-mapreduce-client-hs-plugins
|_____hadoop-mapreduce-client-nativetask
|_____hadoop-mapreduce-client-uploader

```

从第三章开始按照上午模块顺序对各个模块的源码进行逐一阅读分析。