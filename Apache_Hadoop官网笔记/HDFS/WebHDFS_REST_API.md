# WebHDFS REST API

 ## 1. 介绍

HTTP REST API支持HDFS的完整FileSystem/FileContext接口。下面介绍API以及相应FileSystem/FileContext方法。

### 1.1 操作

- HTTP GET
	- [`OPEN`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Open_and_Read_a_File) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).open)
	- [`GETFILESTATUS`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Status_of_a_FileDirectory) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).getFileStatus)
	- [`LISTSTATUS`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#List_a_Directory) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).listStatus)
	- [`LISTSTATUS_BATCH`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Iteratively_List_a_Directory) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).listStatusIterator)
	- [`GETCONTENTSUMMARY`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Get_Content_Summary_of_a_Directory) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).getContentSummary)
	- [`GETQUOTAUSAGE`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Get_Quota_Usage_of_a_Directory) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).getQuotaUsage)
	- [`GETFILECHECKSUM`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Get_File_Checksum) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).getFileChecksum)
	- [`GETHOMEDIRECTORY`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Get_Home_Directory) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).getHomeDirectory)
	- [`GETDELEGATIONTOKEN`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Get_Delegation_Token) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).getDelegationToken)
	- [`GETTRASHROOT`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Get_Trash_Root) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).getTrashRoot)
	- [`GETXATTRS`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Get_an_XAttr) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).getXAttr)
	- [`GETXATTRS`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Get_multiple_XAttrs) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).getXAttrs)
	- [`GETXATTRS`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Get_all_XAttrs) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).getXAttrs)
	- [`LISTXATTRS`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#List_all_XAttrs) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).listXAttrs)
	- [`CHECKACCESS`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Check_access) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).access)
	- [`GETALLSTORAGEPOLICY`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Get_all_Storage_Policies) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).getAllStoragePolicies)
	- [`GETSTORAGEPOLICY`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Get_Storage_Policy) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).getStoragePolicy)
	- [`GETSNAPSHOTDIFF`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Get_Snapshot_Diff)
	- [`GETSNAPSHOTTABLEDIRECTORYLIST`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Get_Snapshottable_Directory_List)
	- [`GETFILEBLOCKLOCATIONS`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Get_File_Block_Locations) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).getFileBlockLocations)
	- [`GETECPOLICY`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Get_EC_Policy) (see [HDFSErasureCoding](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HDFSErasureCoding.html#Administrative_commands).getErasureCodingPolicy)
- HTTP PUT
	- [`CREATE`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Create_and_Write_to_a_File) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).create)
	- [`MKDIRS`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Make_a_Directory) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).mkdirs)
	- [`CREATESYMLINK`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Create_a_Symbolic_Link) (see [FileContext](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileContext.html).createSymlink)
	- [`RENAME`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Rename_a_FileDirectory) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).rename)
	- [`SETREPLICATION`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Set_Replication_Factor) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).setReplication)
	- [`SETOWNER`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Set_Owner) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).setOwner)
	- [`SETPERMISSION`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Set_Permission) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).setPermission)
	- [`SETTIMES`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Set_Access_or_Modification_Time) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).setTimes)
	- [`RENEWDELEGATIONTOKEN`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Renew_Delegation_Token) (see [DelegationTokenAuthenticator](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/security/token/delegation/web/DelegationTokenAuthenticator.html).renewDelegationToken)
	- [`CANCELDELEGATIONTOKEN`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Cancel_Delegation_Token) (see [DelegationTokenAuthenticator](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/security/token/delegation/web/DelegationTokenAuthenticator.html).cancelDelegationToken)
	- [`CREATESNAPSHOT`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Create_Snapshot) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).createSnapshot)
	- [`RENAMESNAPSHOT`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Rename_Snapshot) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).renameSnapshot)
	- [`SETXATTR`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Set_XAttr) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).setXAttr)
	- [`REMOVEXATTR`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Remove_XAttr) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).removeXAttr)
	- [`SETSTORAGEPOLICY`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Set_Storage_Policy) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).setStoragePolicy)
	- [`ENABLEECPOLICY`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Enable_EC_Policy) (see [HDFSErasureCoding](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HDFSErasureCoding.html#Administrative_commands).enablePolicy)
	- [`DISABLEECPOLICY`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Disable_EC_Policy) (see [HDFSErasureCoding](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HDFSErasureCoding.html#Administrative_commands).disablePolicy)
	- [`SETECPOLICY`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Set_EC_Policy) (see [HDFSErasureCoding](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HDFSErasureCoding.html#Administrative_commands).setErasureCodingPolicy)
- HTTP POST
	- [`APPEND`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Append_to_a_File) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).append)
	- [`CONCAT`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Concat_Files) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).concat)
	- [`TRUNCATE`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Truncate_a_File) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).truncate)
	- [`UNSETSTORAGEPOLICY`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Unset_Storage_Policy) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).unsetStoragePolicy)
	- [`UNSETECPOLICY`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Unset_EC_Policy) (see [HDFSErasureCoding](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/HDFSErasureCoding.html#Administrative_commands).unsetErasureCodingPolicy)
- HTTP DELETE
	- [`DELETE`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Delete_a_FileDirectory) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).delete)
	- [`DELETESNAPSHOT`](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Delete_Snapshot) (see [FileSystem](https://hadoop.apache.org/docs/stable/api/org/apache/hadoop/fs/FileSystem.html).deleteSnapshot)

### 1.2 文件系统URI与HTTP URL

WebHDFS的文件系统schema为`webhdfs://`。WebHDFS文件系统URI具有以下格式：

```
	webhdfs://<HOST>:<HTTP_PORT>/<PATH>
```

上面的WebHDFS URI对应于下面的HDFS URI：

```
  hdfs://<HOST>:<RPC_PORT>/<PATH>
```

在REST API中，前缀`/webhdfs/v1`插入路径中，并在末尾附加查询。对应的HTTP URL具有以下格式：

```
  http://<HOST>:<HTTP_PORT>/webhdfs/v1/<PATH>?op=...
```

注意：如果WebHDFS受SSL保护，则Schema应为`swebhdfs://`：

```
  swebhdfs://<HOST>:<HTTP_PORT>/<PATH>
```

### 1.3 HDFS配置选项

以下是WebHDFS的HDFS配置选项。

| 选项名称                                  | 描述                                                         |
| ----------------------------------------- | ------------------------------------------------------------ |
| dfs.web.authentication.kerberos.principal | Hadoop-Auth在HTTP端点中使用的HTTP Kerberos主体。 根据Kerberos HTTP SPNEGO规范，HTTP Kerberos主体必须以`HTTP/`开头。值“ *”在密钥表中找到的所有HTTP主体 |
| dfs.web.authentication.kerberos.keytab    | Kerberos密钥表文件，其中包含HTTP端点中Hadoop-Auth使用的HTTP Kerberos主体的凭据 |
| dfs.webhdfs.socket.connect-timeout        | 失败之前等待建立连接的时间。 指定为持续时间，即数值后跟一个单位符号，例如2m，持续2分钟。 默认为60秒 |
| dfs.webhdfs.socket.read-timeout           | 失败之前等待数据到达的时间。 默认为60秒                      |

## 2. 认证方式

关闭安全模式后，已认证的用户就是user.name查询参数中指定的用户名。 如果未设置user.name参数，则服务器可以将通过身份验证的用户设置为默认的Web用户(如果有)，或者返回错误响应。

启用安全模式后，将通过Hadoop委托令牌或Kerberos SPNEGO执行身份验证。 如果在委托查询参数中设置了令牌，则经过身份验证的用户就是令牌中编码的用户。 如果未设置委托参数，则通过Kerberos SPNEGO对用户进行身份验证。

以下是使用curl命令工具的示例：

- 当安全模式关闭后

	```
	curl -i "http://<HOST>:<PORT>/webhdfs/v1/<PATH>?[user.name=<USER>&]op=..."
	```

- 启用安全模式后，使用Kerberos SPNEGO进行身份验证：

	```
	curl -i --negotiate -u : "http://<HOST>:<PORT>/webhdfs/v1/<PATH>?op=..."
	```

- 启用安全模式后，使用Hadoop委托令牌进行身份验证：

	```
	curl -i "http://<HOST>:<PORT>/webhdfs/v1/<PATH>?delegation=<TOKEN>&op=..."
	```

另外，WebHDFS在客户端支持OAuth2。 Namenode和Datanode当前不支持使用OAuth2的客户端，但其他实现WebHDFS REST接口的后端可能支持。

默认情况下，WebHDFS支持两种类型的OAuth2代码授予方式(用户提供的刷新和访问令牌或用户提供的凭据)，并提供可插拔的机制来根据OAuth2 RFC实施其他OAuth2身份验证或自定义身份验证。 使用提供的任何一种代码授予机制时，WebHDFS客户端将根据需要刷新访问令牌。

仅应为未运行Kerberos SPENGO的客户端启用OAuth2。

| OAuth2代码授予机制 | 描述                                                         | dfs.webhdfs.oauth2.access.token.provider值                   |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 权限码授予         | 用户提供初始访问令牌和刷新令牌，然后分别用于验证WebHDFS请求和获取替换访问令牌 | org.apache.hadoop.hdfs.web.oauth2.ConfRefreshTokenBasedAccessTokenProvider |
| 客户证书授予       | 用户提供用于获取访问令牌的凭据，然后将其用于对WebHDFS请求进行身份验证 | org.apache.hadoop.hdfs.web.oauth2.ConfCredentialBasedAccessTokenProvider |

## 3. SWebHDFS的SSL配置

要使用SWebHDFS FileSystem(即使用swebhdfs协议)，需要在客户端上指定SSL配置文件。 必须指定3个参数：

| SSL 属性                       | 描述                                         |
| ------------------------------ | -------------------------------------------- |
| ssl.client.truststore.location | 可信任文件的存储位置，其中包含NameNode的证书 |
| ssl.client.truststore.type     | 信任文件的格式                               |
| ssl.client.truststore.password | 信任文件的密码                               |

以下是示例SSL配置文件(ssl-client.xml)：

```xml
<configuration>
  <property>
    <name>ssl.client.truststore.location</name>
    <value>/work/keystore.jks</value>
    <description>Truststore to be used by clients. Must be specified.</description>
  </property>

  <property>
    <name>ssl.client.truststore.password</name>
    <value>changeme</value>
    <description>Optional. Default value is "".</description>
  </property>

  <property>
    <name>ssl.client.truststore.type</name>
    <value>jks</value>
    <description>Optional. Default value is "jks".</description>
  </property>
</configuration>
```

SSL配置文件必须在客户端程序的类路径中，并且文件名需要在core-site.xml中指定：

```xml
<property>
  <name>hadoop.ssl.client.conf</name>
  <value>ssl-client.xml</value>
  <description>
    Resource file from which ssl client keystore information will be extracted.
    This file is looked up in the classpath, typically it should be in Hadoop
    conf/ directory. Default value is "ssl-client.xml".
  </description>
</property>
```

## 4. 代理用户

启用代理用户功能后，代理用户P可以代表另一个用户U提交请求。必须在doas查询参数中指定U的用户名，除非在身份验证中提供了委派令牌。 在这种情况下，用户P和U的信息都必须编码在委托令牌中。