# Hadoop源码分析前言

## 1. 源码下载

本人下载的Hadoop版本是3.2.1，源码下载链接：

```sh
https://mirrors.tuna.tsinghua.edu.cn/apache/hadoop/common/hadoop-3.2.1/hadoop-3.2.1-src.tar.gz
```

下载后解压到指定目录即可。

## 2. 源码导入

可以有多个软件支持Hadoop源码导入与阅读，本人使用的IDEA，IDEA导入的步骤可以从网上搜到，在此就不做深入讨论了。

## 3. Hadoop项目目录概览

等待IDEA下载Hadoop依赖包和导入源码完毕后，即可看见Hadoop整体项目架构：

```
hadoop-3.2.1-src Hadoop导入时的项目名称，可以修改，默认hadoop-x.y.z-src
|_____dev-support
|_____hadoop-assemblies  Hadoop项目的打包方式
|_____hadoop-build-tools  Hadoop项目编译工具模块
|_____hadoop-client-modules Hadoop客户端模块
|_____hadoop-cloud-storage-project
|_____hadoop-common-project
|_____hadoop-dist
|_____hadoop-hdfs-project
|_____hadoop-mapreduce-project  Hadoop MapReduce计算框架(核心模块)	  
|_____hadoop-maven-plugins
|_____hadoop-minicluster
|_____hadoop-project
|_____hadoop-project-dist
|_____hadoop-tools
|_____hadoop-yarn-project
```

推荐先从几个重点的模块开始阅读，例如：MapReduce、YARN、HDFS、Core、Client。

下面就开始在源码的海洋中颤抖吧！！！