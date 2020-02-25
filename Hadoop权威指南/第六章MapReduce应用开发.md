# 第六章 MapReduce应用开发

本章从实现层面介绍Hadoop中开发MapReduce程序。MapReduce编程遵循一个特征流程：

- 编写map函数与reduce函数；
- 编写mrUnit单元测试，确保map函数和reduce函数的运行符合预期；
- 编写驱动器程序来运行作业；
- 使用本地IDEA中的小的本地数据集运行并调试；
- 程序按照预期运行则打包部署在集群运行。在集群上运行调试程序具有一定难度，可以通过一些常用技术使其变得简单；
- 对程序优化调整，加快MapReduce运行速度，需要执行一些标准检查；
- 进行任务剖析(task profiling)，分布式程序的分析并不简单，Hadoop提供了钩子(hook)来辅助分析过程。

在开发MR程序前，需要设置和配置开发环境。

## 6.1 用于配置的API

Hadoop中的组件是通过Hadoop自己的配置API来配置的，org.apache.hadoop.conf包中的Confrguration类代表配置属性及其取值集合，每个属性由一个String来命名，值的类型可以是Java基本类型(如boolean、int、boolean)，其他类型(如：Class、java.io.File)以及String集合。

Configuration从资源文件中读取其属性值。参见例6-1:

**例6-1 configuration-1.xml**

```xml
<?xml version="1.0"?>
<configuration>
    <property>
        <name>color</name>
        <value>yellow</value>
    </property>
    <property>
        <name>size</name>
        <value>10</value>
    </property>
    <property>
        <name>weight</name>
        <value>heavy</value>
        <final>true</final>
      	<decription>weight</decription>
    </property>
    <property>
        <name>size-weight</name>
        <value>${size},${weight}</value>
    </property>
</configuration>
```

使用上述配置文件：

```java
Configuration conf = new Configuration();
conf.addReasource("configuration-1.xml");
assertThat(conf.get("color"), is("yellow"));
assertThat(conf.getInt("size", 0), is(10));
assertThat(conf.get("breadth", "wide"), is("wide")); // get()方法允许为XML文件中没有定义的属性指定默认值
```

### 6.1.1 资源合并

使用多个资源文件来定义一个Configuration时，后来添加到资源文件的属性会覆盖之前定义的属性，但是被标记为**final**的属性不能被后面的定义所覆盖。在Hadoop中，用于分离系统默认属性(core-default.xml内部定义的属性)与(core-site.xml文件定义的属性)位置相关的覆盖属性。

### 6.1.2 扩展变量

配置属性可以用其他属性或其他属性进行定义，例如文件`configuration-1.xml`中`size-weight`属性可以定义为`${size}`和`${weight}`,而且这些属性是用配置文件中的值来扩展的:

```java
conf.get("size-weight"), is("12,heavy")
```

系统属性的优先级高于资源文件中定义的属性，该特性特别适用于命令行方式下使用JVM参数-Dproperty=value来覆盖属性。

```java
System.setProperty("size", "14");
assertThat(conf.get("size-weight"), is("14,heavy"));
```

**注意：**虽然配置属性可以通过系统属性来定义,但除非系统属性使用配置属性重新定义,否则他们是无法通过配置API进行访问的，即系统属性重新定义的配置属性才能通过Configuration API访问

```java
System.setProperty("length", "2");
assertThat(conf.get("length"), is(String)null);
```

## 6.2  开发环境配置

新建Maven项目，在POM中定义编译和测试Map-Reduce程序所需依赖，如例6-3所示

**例6-3 编译和测试MapReduce应用的Maven POM**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.bovenson</groupId>
    <artifactId>mapreduce</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
        <hadoop.version>2.6.1</hadoop.version>
    </properties>

    <dependencies>
        <!-- Hadoop main artifact -->
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-core</artifactId>
            <version>${hadoop.version}</version>
        </dependency>
        <!-- Unit test artifacts -->
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.10</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.hamcrest</groupId>
            <artifactId>hamcrest-all</artifactId>
            <version>1.1</version>
            <!--<scope>test</scope>-->
        </dependency>
        <!-- Hadoop test artifacts for running mini clusters -->
        <dependency>
            <groupId>org.apache.mrunit</groupId>
            <artifactId>mrunit</artifactId>
            <version>1.1.0</version>
            <classifier>hadoop2</classifier>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-test</artifactId>
            <version>1.0.0</version>
            <<scope>test</scope>
        </dependency>
  			<!-- Hadoop test artifact running mini clusters-->
  			<dependency>
            <groupId>org.apache.hadoop</groupId>
            <artifactId>hadoop-minicluser</artifactId>
            <version>${hadoop.version}</version>
        </dependency>
        <!-- Missing dependency for running mini clusters -->
        <dependency>
            <groupId>com.sun.jersey</groupId>
            <artifactId>jersey-core</artifactId>
            <version>1.8</version>
            <!--<scope>test</scope>-->
        </dependency>
    </dependencies>

    <build>
        <finalName>hadoop-max-temperature-text</finalName>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <!--<version>2.3.2</version>-->
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-jar-plugin</artifactId>
                <!--<version>2.4</version>-->
                <configuration>
                    <outputDirectory>${basedir}</outputDirectory>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

- hadoop-client：包含了HDFS和MapReduce交互需要的所有Hadoop client-side类，用于构建MapReduce Job
- junit及两个辅助库：运行单元测试
- mrunit：用于写MapReduce测试
- Hadoop-minicluster：包含"mini-"集群，在单个JVM中运行Hadoop集群进行测试

### 6.2.1 管理配置

开发Hadoop应用时，经常需要在本地运行和集群运行之间进行切换，为了进行环境切换，常用的一种方法是：Hadoop的配置文件包含每个集群的连接配置，在运行Hadoop医用或工具时指定连接配置。Hadoop的配置文件最好放在Hadoop安装目录外，以便于轻松在Hadoop不同版本之间进行切换，从而避免重复和配置文件丢失。

假设conf目录有如下三个配置：hadoop-local.xml、hadoop-localhost.xml、hadoop-cluster.xml。

**hadoop-local.xml**：使用默认Hadoop配置

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>file:///</value>
    </property>

    <property>
        <name>mapreduce.framework.name</name>
        <value>local</value>
    </property>
</configuration>
```

**hadoop-localhost.xml**：配置指向本地主机上运行的namenode和YARN

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost</value>
    </property>

    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>

    <property>
        <name>yarn.resourcemanager.address</name>
        <value>localhost:8032</value>
    </property>
</configuration>
```

**Hadoop-cluster.xml**：集群上namenode和YARN的详细信息

```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://namenode:9000</value>
    </property>

    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>

    <property>
        <name>yarn.resourcemanager.address</name>
        <value>resourcenode:8032</value>
    </property>
</configuration>
```

使用-conf命令行来使用各种配置，

```sh
hadoop jar xxx.jar -conf conf/hadoop-cluster.xml
hadoop jar xxx.jar -conf conf/hadoop-local.xml...
hadoop -fs -conf conf/hadoop-localhost.xml -ls .
```

缺省-conf选项，Hadoop默认从`${HADOOP_HOME/etc/hadoop}`下读取Hadoop配置信息，如果指定了`${HADOOP_CONF_DIR}`，则从执行配置文件目录读取。

**Tips 管理配置方法**

将etc/hadoop目录从Hadoop的安装位置拷贝至另一个位置，并将`*-site.xml`配置文件也放置在该位置，将`${HADOOP_CONF_DIR}`环境变量设置为该文件路径。这种管理方式的优点是不必要每个命令执行-conf。

Hadoop自带工具支持-conf选项，可以直接用程序(例如运行MapReduce作业的程序)通过使用Tool接口来支持-conf选项。

### 6.2.2 辅助类GenericOptionsParser、Tool和ToolRunner

为了简化命令行方式运行作业，Hadoop自带一些辅助类。GenericOptionsParse用来解释常用的Hadoop命令行选项，并根据需要，为Configuration对象设置相应的取值。通产不直接使用GenericOptionParser对象，可以直接实现Tool 类，因为Tool内部其实就是使用了GenericOptionParser 类，Tool类继承自Configurable类

```java
public interface Tool extends Configurable {
		int run(String [] args) throws Exception;
}
```

范例6-4 Tool实现打印Configuration对象的属性

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.conf.Configured;
import org.apache.hadoop.util.Tool;
import org.apache.hadoop.util.ToolRunner;

import java.util.Map;

public class ConfigurationPrinter extends Configured implements Tool {

    static
    {
        Configuration.addDefaultResource("hdfs-default.xml");
      	Configuration.addDefaultResource("hdfs-site.xml");
     		Configuration.addDefaultResource("yarn-default.xml");
      	Configuration.addDefaultResource("yarn-site.xml");
      	Configuration.addDefaultResource("mapred-default.xml");
      	Configuration.addDefaultResource("mapred-site.xml");
    }
    @Override
    public int run(String[] args) throws Exception {
        //这里可以直接调用getConf() 方法是因为：getConf()是从Configured 中继承来的
        //而 Configured 的方法是从 Configurable 中实现而来的。
        Configuration conf = getConf();
        for (Map.Entry<String, String> entry : conf) {
            System.out.printf("%s=%s\n", entry.getKey(), entry.getValue());
        }
        return 0;
    }

    public static void main(String[] args) throws Exception {
        int exitCode = ToolRunner.run(new ConfigurationPrinter(), args);
        System.exit(exitCode);
    }
}
```

所有的 `Tool`的实现类同时需要实现 `Configurable`类 (因为Tool 继承了该类)。其子类`Configured`是最简单的实现方式，`run()`方法通过`Configurable`的 `getConf()`方法获取`Configuration`。

但是代码里又用到了`ToolRunner` 类：

```java
/**
   * Runs the given <code>Tool</code> by {@link Tool#run(String[])}, after 
   * parsing with the given generic arguments. Uses the given 
   * <code>Configuration</code>, or builds one if null.
   * 
   * Sets the <code>Tool</code>'s configuration with the possibly modified 
   * version of the <code>conf</code>.  
   * 
   * @param conf <code>Configuration</code> for the <code>Tool</code>.
   * @param tool <code>Tool</code> to run.
   * @param args command-line arguments to the tool.
   * @return exit code of the {@link Tool#run(String[])} method.
   */
  public static int run(Configuration conf, Tool tool, String[] args) 
  throws Exception{
    if(conf == null) {
      conf = new Configuration();
    }
    GenericOptionsParser parser = new GenericOptionsParser(conf, args);
    //set the configuration back, so that Tool can configure itself
    tool.setConf(conf);

    //get the args w/o generic hadoop args
    String[] toolArgs = parser.getRemainingArgs();
    return tool.run(toolArgs);
}

/**
   * Runs the <code>Tool</code> with its <code>Configuration</code>.
   * 
   * Equivalent to <code>run(tool.getConf(), tool, args)</code>.
   * 
   * @param tool <code>Tool</code> to run.
   * @param args command-line arguments to the tool.
   * @return exit code of the {@link Tool#run(String[])} method.
   */
  public static int run(Tool tool, String[] args) 
    throws Exception{
    return run(tool.getConf(), tool, args);
  }
```

可以看到这个 `run()`其实底层调用了 另一个`run()`方法， 但是在运行之前添加了一个`tool.getConf()`参数，这个`getConf()`方法是用于得到一个`Configuration`实例，从而传递给`run()`方法作为参数。