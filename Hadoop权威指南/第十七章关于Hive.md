# 第十七章 关于Hive

HIve是一个构建在Hadoop上的数据仓库框架，其设计目的是让精通SQL技能但是编程较弱的数据分析师能够对存放在HDFS中的大规模数据集执行查询。目前，很多组织将Hive作为一个通用的，可伸缩的数据处理平台。

SQL适用于分析任务，不适合用来开发复杂的机器学习算法。其特点如下：

1. 简单、容易上手 (提供了类似 sql 的查询语言 hql)，使得精通 sql 但是不了解 Java 编程的人也能很好地进行大数据分析；
2. 灵活性高，可以自定义用户函数 (UDF) 和存储格式；
3. 为超大的数据集设计的计算和存储能力，集群扩展容易;
4. 统一的元数据管理，可与 presto／impala／sparksql 等共享数据；
5. 执行延迟高，不适合做数据的实时处理，但适合做海量数据的离线处理。

## 17.1 安装Hive

Hive一般在工作站上运行，把SQL查询站换位一系列在Hadoop集群上运行的作业，Hive把数据组织为表，通过这种方式为存储在HDFS上的数据赋予结构，元数据存储在metastore数据库中。

安装Hive非常简单，首先必须在本地安装和集群上相同版本的Hadoop。

**Tips Hive能和哪些版本的Hadoop共同工作**

每个Hive的发布版本都会被设计为能够和多个版本的Hadoop共同工作，一般而言，Hive支持Hadoop最新发布的稳定版本以及之前的老版本。

**1. 下载解压Hive**

下载所需版本的 Hive，这里我下载版本为 `cdh5.15.2`。下载地址：http://archive.cloudera.com/cdh5/cdh/5/

```shell
# 下载后进行解压
tar -zxvf hive-1.1.0-cdh5.15.2.tar.gz
```

**2. 配置环境变量**

```shell
vim /etc/profile
```

添加环境变量：

```shell
export HIVE_HOME=/usr/app/hive-1.1.0-cdh5.15.2
export PATH=$HIVE_HOME/bin:$PATH
```

使得配置的环境变量立即生效：

```shell
source /etc/profile
```

**3. 修改配置**

**(1) hive-env.sh**

进入安装目录下的 `conf/` 目录，拷贝 Hive 的环境配置模板 `flume-env.sh.template`

```shell
cp hive-env.sh.template hive-env.sh
```

修改 `hive-env.sh`，指定 Hadoop 的安装路径：

```shell
HADOOP_HOME=${your HADOOP_HOME}
```

**(2) hive-site.xml**

新建 hive-site.xml 文件，内容如下，主要是配置存放元数据的 MySQL 的地址、驱动、用户名和密码等信息：

```xml
<?xml version="1.0"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>

<configuration>
  <property>
    <name>javax.jdo.option.ConnectionURL</name>
    <value>jdbc:mysql://hadoop001:3306/hadoop_hive?createDatabaseIfNotExist=true</value>
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

**4. 拷贝数据库驱动**

将 MySQL 驱动包拷贝到 Hive 安装目录的 `lib` 目录下, MySQL 驱动的下载地址为：https://dev.mysql.com/downloads/connector/j/ , 在本仓库的[resources](https://github.com/heibaiying/BigData-Notes/tree/master/resources) 目录下我也上传了一份，有需要的可以自行下载。

**5. 初始化元数据库**

- 当使用的 hive 是 1.x 版本时，可以不进行初始化操作，Hive 会在第一次启动的时候会自动进行初始化，但不会生成所有的元数据信息表，只会初始化必要的一部分，在之后的使用中用到其余表时会自动创建；

- 当使用的 hive 是 2.x 版本时，必须手动初始化元数据库。初始化命令：

	```shell
	# schematool 命令在安装目录的 bin 目录下，由于上面已经配置过环境变量，在任意位置执行即可
	schematool -dbType mysql -initSchema
	```

这里我使用的是 CDH 的 `hive-1.1.0-cdh5.15.2.tar.gz`，对应 `Hive 1.1.0` 版本，可以跳过这一步。

**6. 启动**

由于已经将 Hive 的 bin 目录配置到环境变量，直接使用以下命令启动，成功进入交互式命令行后执行 `show databases` 命令，无异常则代表搭建成功。

```shell
hive
```

在 Mysql 中也能看到 Hive 创建的库和存放元数据信息的表

![](./img/hive-mysql-tables.jpg)

### Hive的Shell环境

Hive的shell环境是用户和Hive交互，发出HiveSQL命令的主要方式。HiveQL是HIve的查询语言。

第一次启动Hive时，可以通过列出Hive的表来检查Hive是否正常工作

```shell
hive > show tables;
OK
Time token: 0.473 seconds
```

![](./img/hive-start.jpg)

使用-f选项可以运行指定文件的命令。

```hive
hive -f xxx.sql
```

相对于较短的脚本，可以使用-e选项在行内嵌入命令，此时不需要结束的分号。

```shell
hive -e 'select * from table'
```

## 17.2 示例

使用Hive查询数据集，步骤如下：

1. 把数据加载到Hive管理的存储，可以是本地，也可以是HDFS。

	```sql
	CREATE TABLE records (year STRING, temperature INT, quality INT)
	ROW FORMATE DELIMITED
		FILEDS TERMINATED BY '\t';
	```

	第一行声明一个records表，包含三列：year、temperature、quality，必须指明每列数据类型。第二行ROW FORMATE声明数据文件的每一行是由制表符分隔的文本，Hive按照这一格式读取数据：每行三个字段，分别对应于表中的三列，字段间使用制表符分割，每行以换行符分割。

2. 向Hive输入数据

	```sql
	load data local inpath "/usr/file/emp.txt" overwrite into table emp;
	```

	这一指令表示将指定的本地温江放入其仓库目录中，这只是个简单的文件系统操作，并不解析文件或把他存储为内部数据库格式，因为Hive并不强制使用人格特定文件格式，文件以原样逐字存储。

	在Hive的仓库目录中，表存储为目录，仓库目录由选项hive.metastore.warehouse.dir控制，默认值为：/usr/hive/warehouse。

3. 使用HQL语句操作数据

	```sql
	hive> select year, MAX(tempereture) from records where temperature != 9999 and quality in (0,1,4,5,9) Group by year;
	```

## 17.3 运行Hive

