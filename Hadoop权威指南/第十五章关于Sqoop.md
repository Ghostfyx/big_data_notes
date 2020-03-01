# 第十七章 关于Sqoop

Hadoop平台的最大优势在于它支持不同形式的数据，HDFS能够可靠地存储日志和来自不同渠道的数据，MaoReduce程序能够分析多种数据格式，抽取相关信息并将多个数据集组合成非常有用的结果。

通常，一个组织中有价值的数据都存储在关系型数据库(RDBMS)等结构化存储器中，Apache Sqoop允许用户将数据从结构化存储器抽取到Hadoop中，用于进一步处理。抽取的数据可以被MapReduce程序使用，可以被类似于Hive的工具使用，完成最终结果分析，Sqoop便可以将这些结果导回数据存储器。

- 导入数据：从 MySQL，Oracle 等关系型数据库中导入数据到 HDFS、Hive、HBase 等分布式文件存储系统中；
- 导出数据：从 分布式文件系统中导出数据到关系数据库中。

其原理是将执行命令转化成 MapReduce 作业来实现数据的迁移，如下图：

![](./img/sqoop-tool.jpg)

## 15.1 获取与安装Sqoop

版本选择：目前 Sqoop 有 Sqoop 1 和 Sqoop 2 两个版本，但是截至到目前，官方并不推荐使用 Sqoop 2，因为其与 Sqoop 1 并不兼容，且功能还没有完善，所以这里优先推荐使用 Sqoop1。

![](./img/sqoop-version-selected.jpg)

### 15.1.1 下载并解压

下载所需版本的 Sqoop ，这里我下载的是 `CDH` 版本的 Sqoop 。下载地址为：http://archive.cloudera.com/cdh5/cdh/5/

```sh
# 下载后进行解压
tar -zxvf  sqoop-1.4.6-cdh5.15.2.tar.gz
```

### 15.1.2 配置环境变量

```sh
# vim /etc/profile
```

添加环境变量：

```sh
export SQOOP_HOME=/usr/local/sqoop/sqoop-1.4.6-cdh5.15.2
export PATH=$SQOOP_HOME/bin:$PATH
```

使得配置的环境变量立即生效：

```sh
# source /etc/profile
```

###15.1.3 修改配置

进入安装目录下的 `conf/` 目录，拷贝 Sqoop 的环境配置模板 `sqoop-env.sh.template`

```sh
cp sqoop-env-template.sh sqoop-env.sh
```

修改 `sqoop-env.sh`，内容如下 (以下配置中 `HADOOP_COMMON_HOME` 和 `HADOOP_MAPRED_HOME` 是必选的，其他的是可选的)：

```sh
# Set Hadoop-specific environment variables here.
#Set path to where bin/hadoop is available
export HADOOP_COMMON_HOME=/usr/local/hadoop/hadoop-2.6.0-cdh5.15.2

#Set path to where hadoop-*-core.jar is available
export HADOOP_MAPRED_HOME=/usr/local/hadoop/hadoop-2.6.0-cdh5.15.2

#set the path to where bin/hbase is available
export HBASE_HOME=/usr/local/hbase/hbase-1.2.0-cdh5.15.2

#Set the path to where bin/hive is available
export HIVE_HOME=/usr/local/hive/hive-1.1.0-cdh5.15.2

#Set the path for where zookeper config dir is
export ZOOCFGDIR=/usr/local/zookeeper/zookeeper-3.4.13/conf
```

### 15.1.4 拷贝数据库驱动

将 MySQL 驱动包拷贝到 Sqoop 安装目录的 `lib` 目录下, 驱动包的下载地址为 https://dev.mysql.com/downloads/connector/j/ 。

### 15.1.5 验证

由于已经将 sqoop 的 `bin` 目录配置到环境变量，直接使用以下命令验证是否配置成功：

```sh
sqoop version
```

出现对应的版本信息则代表配置成功：

![](./img/sqoop-version.jpg)

这里出现的两个 `Warning` 警告是因为我们本身就没有用到 `HCatalog` 和 `Accumulo`，忽略即可。Sqoop 在启动时会去检查环境变量中是否有配置这些软件，如果想去除这些警告，可以修改 `bin/configure-sqoop`，注释掉不必要的检查。

```sh
# Check: If we can't find our dependencies, give up here.
if [ ! -d "${HADOOP_COMMON_HOME}" ]; then
  echo "Error: $HADOOP_COMMON_HOME does not exist!"
  echo 'Please set $HADOOP_COMMON_HOME to the root of your Hadoop installation.'
  exit 1
fi
if [ ! -d "${HADOOP_MAPRED_HOME}" ]; then
  echo "Error: $HADOOP_MAPRED_HOME does not exist!"
  echo 'Please set $HADOOP_MAPRED_HOME to the root of your Hadoop MapReduce installation.'
  exit 1
fi

## Moved to be a runtime check in sqoop.
if [ ! -d "${HBASE_HOME}" ]; then
  echo "Warning: $HBASE_HOME does not exist! HBase imports will fail."
  echo 'Please set $HBASE_HOME to the root of your HBase installation.'
fi

## Moved to be a runtime check in sqoop.
if [ ! -d "${HCAT_HOME}" ]; then
  echo "Warning: $HCAT_HOME does not exist! HCatalog jobs will fail."
  echo 'Please set $HCAT_HOME to the root of your HCatalog installation.'
fi

if [ ! -d "${ACCUMULO_HOME}" ]; then
  echo "Warning: $ACCUMULO_HOME does not exist! Accumulo imports will fail."
  echo 'Please set $ACCUMULO_HOME to the root of your Accumulo installation.'
fi
if [ ! -d "${ZOOKEEPER_HOME}" ]; then
  echo "Warning: $ZOOKEEPER_HOME does not exist! Accumulo imports will fail."
  echo 'Please set $ZOOKEEPER_HOME to the root of your Zookeeper installation.'
fi
```

### 15.1.6 Sqoop名称

不带参数运行Sqoop没有什么意义，Sqoop组成一组工具或命令，Sqoop help查看所有命令：

```sh
Available commands:
  codegen            Generate code to interact with database records
  create-hive-table  Import a table definition into Hive
  eval               Evaluate a SQL statement and display the results
  export             Export an HDFS directory to a database table
  help               List available commands
  import             Import a table from a database to HDFS
  import-all-tables  Import tables from a database to HDFS
  import-mainframe   Import datasets from a mainframe server to HDFS
  job                Work with saved jobs
  list-databases     List available databases on a server
  list-tables        List available tables in a database
  merge              Merge results of incremental imports
  metastore          Run a standalone Sqoop metastore
  version            Display version information

See 'sqoop help COMMAND' for information on a specific command.
```

Sqoop help import查看导入数据命令详情：

```sh
Command arguments:
	--connect <jdbc-url> Specify JDBC connect String
	--driver <class-name> Manally specify JDBC driver class to use
	--hadoop-home <dir> Override $HADOOP_HOME
	--help 
	-p Read password from console
	--password <password> Set authentication password
	--username <username> Set authentication username
	--verbose
	...
```

## 15.2 Sqoop连接器

Sqoop拥有一个可扩展的框架，使得它能够从(向)任何支持批量数据传输的外部存储系统导入(导出)数据。Sqoop连接器就是这个框架下的一个模块化组件，用于支持Sqoop的导入和导出操作。Sqoop附带的连接器可以支持大多数常用的关系型数据库，包括MySQL、PostgreSQL、Oracle、SQL Server、DB2和Metzza，同时还有一个JDBC连接器，用于连接支持JDBC协议的数据库。

## 15.3 数据导入

在安装了Sqoop之后，可以用它将数据导入到Hadoop。

### 15.3.1 MySQL导入

**1. 查询MySQL所有数据库**

通常用于 Sqoop 与 MySQL 连通测试：

```sh
sqoop list-databases \
--connect jdbc:mysql://hadoop001:3306/ \
--username root \
--password root
```

![](./img/sqoop-list-databases.jpg)

**2. 查询指定数据库中所有数据表**

```sh
sqoop list-tables \
--connect jdbc:mysql://hadoop001:3306/mysql \
--username root \
--password root
```

**3. MySQL数据导入到HDFS**

示例：导出 MySQL 数据库中的 `help_keyword` 表到 HDFS 的 `/sqoop` 目录下，如果导入目录存在则先删除再导入，使用 3 个 `map tasks` 并行导入。

> 注：help_keyword 是 MySQL 内置的一张字典表，之后的示例均使用这张表。

```sh
sqoop import \
--connect jdbc:mysql://hadoop001:3306/mysql \     
--username root \
--password root \
--table help_keyword \           # 待导入的表
--delete-target-dir \            # 目标目录存在则先删除
--target-dir /sqoop \            # 导入的目标目录
--fields-terminated-by '\t'  \   # 指定导出数据的分隔符
-m 3                             # 指定并行执行的 map tasks 数量
```

日志输出如下，可以看到输入数据被平均 `split` 为三份，分别由三个 `map task` 进行处理。数据默认以表的主键列作为拆分依据，如果你的表没有主键，有以下两种方案：

- 添加 `-- autoreset-to-one-mapper` 参数，代表只启动一个 `map task`，即不并行执行；
- 若仍希望并行执行，则可以使用 `--split-by ` 指明拆分数据的参考列。

![](./img/sqoop-map-task.jpg)

**4. 导入验证**

```sh
# 查看导入后的目录
hadoop fs -ls  -R /sqoop
# 查看导入内容
hadoop fs -text  /sqoop/part-m-00000
```

查看 HDFS 导入目录,可以看到表中数据被分为 3 部分进行存储，**这是由指定的并行度决定的**。

![](./img/sqoop_hdfs_ls.jpg)

