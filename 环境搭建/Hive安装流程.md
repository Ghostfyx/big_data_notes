# Linux环境下Hive的安装

[TOC]

## 一、安装Hive

### 1.1 下载并解压

下载所需版本的 Hive，这里我下载版本为 `cdh5.15.2`。下载地址：http://archive.cloudera.com/cdh5/cdh/5/

```sh
# 下载后进行解压
 tar -zxvf hive-1.1.0-cdh5.15.2.tar.gz
```

### 1.2 配置环境变量

```sh
# vim /etc/profile
```

添加环境变量：

```sh
export HIVE_HOME=${your_path}/hive-1.1.0-cdh5.15.2
export PATH=$HIVE_HOME/bin:$PATH
```

使得配置的环境变量立即生效：

```sh
# source /etc/profile
```

### 1.3 修改配置

#### 1. hive_env.sh

进入安装目录下的 `conf/` 目录，拷贝 Hive 的环境配置模板 `flume-env.sh.template`

```
cp hive-env.sh.template hive-env.sh
```

修改 `hive-env.sh`，指定 Hadoop 的安装路径：

```sh
HADOOP_HOME=${your_hadoop_home}
```

#### 2. hive_site.xml

新建 hive-site.xml 文件，内容如下，主要是配置存放元数据的 MySQL 的地址、驱动、用户名和密码等信息：

```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
      <!-- useSSL=false，MySQL在高版本需要指明是否进行SSL连接-->
    <value>jdbc:mysql://{mysql_host}:3306/hadoop_hivecreateDatabaseIfNotExist=true
    &amp;useSSL=false</value>
  </property>
  
  <property>
    <name>javax.jdo.option.ConnectionDriverName</name>
    <value>com.mysql.jdbc.Driver</value>
  </property>
  
  <property>
    <name>javax.jdo.option.ConnectionUserName</name>
    <value>root</value>
  </property>
  
  <property>
    <name>javax.jdo.option.ConnectionPassword</name>
    <value>root</value>
  </property>

</configuration>
```

### 1.4 拷贝数据库驱动

将 MySQL 驱动包拷贝到 Hive 安装目录的 `lib` 目录下, MySQL 驱动的下载地址为：https://dev.mysql.com/downloads/connector/j/ 。

### 1.5 初始化元数据库

- 当使用的 hive 是 1.x 版本时，可以不进行初始化操作，Hive 会在第一次启动的时候会自动进行初始化，但不会生成所有的元数据信息表，只会初始化必要的一部分，在之后的使用中用到其余表时会自动创建；

- 当使用的 hive 是 2.x 版本时，必须手动初始化元数据库。初始化命令：

	```sh
	# schematool 命令在安装目录的 bin 目录下，由于上面已经配置过环境变量，在任意位置执行即可
	schematool -dbType mysql -initSchema
	```

这里使用的是 CDH 的 `hive-1.1.0-cdh5.15.2.tar.gz`，对应 `Hive 1.1.0` 版本，可以跳过这一步。

### 1.6 启动

由于已经将 Hive 的 bin 目录配置到环境变量，直接使用以下命令启动，成功进入交互式命令行后执行 `show databases` 命令，无异常则代表搭建成功。

```sh
hive
```