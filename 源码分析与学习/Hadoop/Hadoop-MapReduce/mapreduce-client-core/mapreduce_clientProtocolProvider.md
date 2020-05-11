# ClientProtocolProvider 源码分析

## 1. ClientProtocolProvider概述

ClientProtocolProvider客户端通信协议提供者的抽象类，在MapReduce中，ClientProtocolProvider抽象类的实现共有YarnClientProtocolProvider、LocalClientProtocolProvider两种，前者为Yarn模式，而后者为Local模式。

## 2. ClientProtocolProvider方法列表

- ```java
	public abstract ClientProtocol create(Configuration conf) throws IOException;
	```

- ```java
	public abstract ClientProtocol create(InetSocketAddress addr,
	    Configuration conf) throws IOException;
	```

- ```java
	public abstract void close(ClientProtocol clientProtocol) throws IOException;
	```

## 3. 抽象类实现

### 3.1 YarnClientProtocolProvider

YARN模式下，由YarnClientProtocolProvider创建提交器YARNRunner。

```java
public class YarnClientProtocolProvider extends ClientProtocolProvider {

  @Override
  public ClientProtocol create(Configuration conf) throws IOException {
    // 获取Job配置的mapreduce.framework.name与MR的YARN_FRAMEWORK_NAME对比
    if (MRConfig.YARN_FRAMEWORK_NAME.equals(conf.get(MRConfig.FRAMEWORK_NAME))) {
      return new YARNRunner(conf);
    }
    return null;
  }

  // 新版本底层调用的是create(conf);方法，InetSocketAddress没有用
  @Override
  public ClientProtocol create(InetSocketAddress addr, Configuration conf)
      throws IOException {
    return create(conf);
  }

  @Override
  public void close(ClientProtocol clientProtocol) throws IOException {
    if (clientProtocol instanceof YARNRunner) {
      ((YARNRunner)clientProtocol).close();
    }
  }
}
```

### 3.2 LocalClientProtocolProvider

用户本地/单机启动，通过`LocalClientProtocolProvider`创建作业提交器`LocalJobRunner`。

Local模式也是需要看参数mapreduce.framework.name的配置是否为local，是的话，返回LocalJobRunner实例，并设置map任务数量为1，否则返回null，值得一提的是，这里参数mapreduce.framework.name未配置的话，默认为local，也就是说，MapReduce需要看参数`mapreduce.framework.name`确定连接模式，但默认是Local模式的。

```java
public class LocalClientProtocolProvider extends ClientProtocolProvider {

  @Override
  public ClientProtocol create(Configuration conf) throws IOException {
     // conf.get(String propertyName, String defaultValue)
      // 初始化framework：取参数mapreduce.framework.name，参数未配置默认为local
      String framework =
        conf.get(MRConfig.FRAMEWORK_NAME, MRConfig.LOCAL_FRAMEWORK_NAME);
    if (!MRConfig.LOCAL_FRAMEWORK_NAME.equals(framework)) {
      return null;
    }
    // 本地模式仅会开启一个map任务
    conf.setInt(JobContext.NUM_MAPS, 1);
    // 返回本地提交器  
    return new LocalJobRunner(conf);
  }

  @Override
  public ClientProtocol create(InetSocketAddress addr, Configuration conf) {
    return null; // LocalJobRunner doesn't use a socket
  }

  @Override
  public void close(ClientProtocol clientProtocol) {
    // no clean up required
  }

}
```

## 4. SPI动态加载

在Cluster类中，Cluster中客户端通信协议ClientProtocol实例，**要么是Yarn模式下的YARNRunner，要么就是Local模式下的LocalJobRunner**。使用`ServiceLoader.load(ClientProtocolProvider.class);`动态加载ClientProtocolProvider实现类：YarnClientProtocolProvider和LocalClientProtocolProvider。

接口全限定名命名的文件位置：

- YarnClientProtocolProvider位于hadoop-mapreduce-client-jobclient模块下的META-INF/services/目录中；
- LocalClientProtocolProvider位于hadoop-mapreduce-client-common模块下的META-INF/services/目录中；