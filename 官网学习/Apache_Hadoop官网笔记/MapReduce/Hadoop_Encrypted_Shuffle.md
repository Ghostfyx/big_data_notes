# Hadoop Encrypted Shuffle

## 1. 简介

 加密shuffle功能允许使用HTTPS和可选的客户端身份验证(也称为双向HTTPS或带有客户端证书的HTTPS)。对MapReduce shuffle过程加密，它包括：

- Hadoop配置，用于在HTTP和HTTPS之间切换
- Hadoop配置，用于指定shuffle和reducers任务获取shuffle数据所使用的密钥库和信任库属性(位置，类型，密码)
- 一种在群集上重新加载信任库的方法(添加或删除节点时)

## 2. 配置

### 2.1 **core-site.xml** 选项

要启用加密shuffle，请在集群中所有节点的core-site.xml中设置以下属性：

| **Property**                         | **Default Value**                                          | **Explanation**                                              |
| :----------------------------------- | :--------------------------------------------------------- | :----------------------------------------------------------- |
| `hadoop.ssl.require.client.cert`     | `false`                                                    | 是否需要客户端证书                                           |
| `hadoop.ssl.hostname.verifier`       | `DEFAULT`                                                  | 提供HttpsURLConnections的主机名验证程序。 有效值为**DEFAULT**, **STRICT**, **STRICT_IE6**, **DEFAULT_AND_LOCALHOST** 和 **ALLOW_ALL** |
| `hadoop.ssl.keystores.factory.class` | `org.apache.hadoop.security.ssl.FileBasedKeyStoresFactory` | The KeyStoresFactory implementation to use                   |
| `hadoop.ssl.server.conf`             | `ssl-server.xml`                                           | Resource file from which ssl server keystore information will be extracted. This file is looked up in the classpath, typically it should be in Hadoop conf/ directory |
| `hadoop.ssl.client.conf`             | `ssl-client.xml`                                           | Resource file from which ssl server keystore information will be extracted. This file is looked up in the classpath, typically it should be in Hadoop conf/ directory |
| `hadoop.ssl.enabled.protocols`       | `TLSv1,SSLv2Hello,TLSv1.1,TLSv1.2`                         | The supported SSL protocols                                  |

重要说明：所有这些属性应在群集配置文件中标记为final。

示例：

```xml
	<property>
    <name>hadoop.ssl.require.client.cert</name>
    <value>false</value>
    <final>true</final>
  </property>

  <property>
    <name>hadoop.ssl.hostname.verifier</name>
    <value>DEFAULT</value>
    <final>true</final>
  </property>

  <property>
    <name>hadoop.ssl.keystores.factory.class</name>
    <value>org.apache.hadoop.security.ssl.FileBasedKeyStoresFactory</value>
    <final>true</final>
  </property>

  <property>
    <name>hadoop.ssl.server.conf</name>
    <value>ssl-server.xml</value>
    <final>true</final>
  </property>

  <property>
    <name>hadoop.ssl.client.conf</name>
    <value>ssl-client.xml</value>
    <final>true</final>
  </property>
```

### 2.2 marred-site.xml属性

| **Property**                    | **Default Value** | **Explanation** |
| :------------------------------ | :---------------- | :-------------- |
| `mapreduce.shuffle.ssl.enabled` | `false`           |                 |

重要说明：所有这些属性应在群集配置文件中标记为final。

示例：

```xml
 <property>
    <name>mapreduce.shuffle.ssl.enabled</name>
    <value>true</value>
    <final>true</final>
  </property>
```

