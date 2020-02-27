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

**范例6-4 Tool实现打印Configuration对象的属性**

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

**Tips 可以设置哪些属性**

ConfigurationPrinter 可以用于了解环境中的某个属性是如何设置的。YARN网络服务器的/conf页面可以查看运行中的守护进程(如：namenode)的配置情况。

Hadoop的默认配置文件在`${HADOOP_HOME}/share/doc$`目录中，包括：core-default.xml、hdfs-default.xml、yarn-default.xml和mappred-default.xml这几个文件，每个属性都有用来解释属性的作用的取值范围。

**表6-1 GenericOptionsParser选项和ToolRunner选项**

| 选项名称                       | 描述                                                         |
| ------------------------------ | ------------------------------------------------------------ |
| -D property=value              | 将指定值赋值给Hadoop配置选项，覆盖配置文件中的默认属性或站点属性，或通过-conf 选项设置的任何属性 |
| -conf filename...              | 将制定文件条件到配置资源列表中，这里设置站点属性或同时设置一组属性的简单方法 |
| -fs uri                        | 用指定的URI设置默认文件系统，是-D fs.default.FS=uri的快捷方式 |
| -jt host:port                  | 用指定主机和端口号设置YARN资源管理器，是-D yarn.resourcemanager.adderss=hostname:port的快捷方式 |
| -files file1,file2,...         | 从本地文件系统或任何指定模式的文件系统中复制指定文件到MapReduce所用文件系统，确保任务工作目录的MR程序可以访问到这些文件 |
| -archives archive1,archive2,.. | 从本地文件系统或任何指定模式的文件系统中复制指定存档到MapReduce所用文件系统，确保任务工作目录的MR程序可以访问到这些存档 |
| -libjars jar1,jar2,..          | 从本地文件系统或任何指定模式的文件系统中复制指定JAR文件到MapReduce所用文件系统，将其加入到MapReduce任务的类路径中，适用于传输作业需要的JAR包，也可以使用maven等打包工具将MR任务需要所有第三方JAR包打入MR任务JAR包。 |



## 6.3 用MRUnit写单元测试

在MapReduce中，map函数和reduce函数的独立测试非常方便，这是由函数风格决定的。MRUnit(http://incubator.apache.org/mrunit/)是一个测试库，它便于将已知的输入传递给mapper或者检查reducer的输出是否符合预期。MRUnit与标准的执行框架(如JUnit)-起使用，因此可以将MapReduce作业的测试作为正常开发环境的一部分运行。

### 6.3.1 关于Mapper

```java
import java.io.IOException;
 
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;
 
public class MaxTemperatureMapper extends Mapper<LongWritable, Text, Text, IntWritable> {
	@Override
	protected void map(LongWritable key, Text value, Mapper<LongWritable, Text, Text, IntWritable>.Context context)
			throws IOException, InterruptedException {
		String line = value.toString();
		String year = line.substring(15, 19);
		int airTemperature;
 
		if (line.charAt(87) == '+') { // parseInt doesn't like leading plus
										// signs
			airTemperature = Integer.parseInt(line.substring(88, 92));
		} else {
			airTemperature = Integer.parseInt(line.substring(87, 92));
		}
 
		String quality = line.substring(92, 93);
		if (airTemperature != MISSING && quality.matches("[01459]")) {
			context.write(new Text(year), new IntWritable(airTemperature));
		}
	}
 
	private static final int MISSING = 9999;
}
```

使用MRUnit进行测试，首先需要创建MapDriver对象，并设置要测试的Mapper类，设定输入、期望输出。具体例子中传递一个天气记录作为mapper的输入，然后检查输出是否是读入的年份和气温。如果没有期望的输出值，MRUnit测试失败。

```java
import java.io.IOException;
 
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Counters;
import org.apache.hadoop.mrunit.mapreduce.MapDriver;
import org.junit.Test;
 
import com.jliu.mr.intro.MaxTemperatureMapper;
 
public class MaxTemperatureMapperTest {
	@Test
	public void testParsesValidRecord() throws IOException {
		Text value = new Text("0043011990999991950051518004+68750+023550FM-12+0382" +
		// ++++++++++++++++++++++++++++++year ^^^^
				"99999V0203201N00261220001CN9999999N9-00111+99999999999");
		// ++++++++++++++++++++++++++++++temperature ^^^^^
		// 由于测试的mapper，所以适用MRUnit的MapDriver
		new MapDriver<LongWritable, Text, Text, IntWritable>()
				// 配置mapper
				.withMapper(new MaxTemperatureMapper())
				// 设置输入值
				.withInput(new LongWritable(0), value)
				// 设置期望输出：key和value
				.withOutput(new Text("1950"), new IntWritable(-11)).runTest();
	}
 
	@Test
	public void testParseMissingTemperature() throws IOException {
		// 根据withOutput()被调用的次数， MapDriver能用来检查0、1或多个输出记录。
		// 在这个测试中由于缺失的温度记录已经被过滤，保证对这种特定输入不产生任何输出
		Text value = new Text("0043011990999991950051518004+68750+023550FM-12+0382" +
		// ++++++++++++++++++++++++++++++Year ^^^^
				"99999V0203201N00261220001CN9999999N9+99991+99999999999");
		// ++++++++++++++++++++++++++++++Temperature ^^^^^
		new MapDriver<LongWritable, Text, Text, IntWritable>()
				.withMapper(new MaxTemperatureMapper())
				.withInput(new LongWritable(0), value)
				.runTest();
	}
}
```

### 6.3.2 关于Reducer

```java
import java.io.IOException;
 
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;
 
public class MaxTemperatureReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
	@Override
	protected void reduce(Text key, Iterable<IntWritable> values,
			Reducer<Text, IntWritable, Text, IntWritable>.Context context) throws IOException, InterruptedException {
 
		int maxValue = Integer.MIN_VALUE;
		for (IntWritable value : values) {
			maxValue = Math.max(maxValue, value.get());
		}
 
		context.write(key, new IntWritable(maxValue));
	}
}
```

对Reducer的测试，与Mapper类似，新建ReducerDriver

``` java
import org.apache.hadoop.mrunit.mapreduce.ReduceDriver;
import java.io.IOException;
import java.util.Arrays;
import org.apache.hadoop.io.*;
import org.junit.Test;
 
import com.jliu.mr.intro.MaxTemperatureReducer;
 
public class MaxTemperatureReducerTest {
	@Test
	public void testRetrunsMaximumIntegerValues() throws IOException {
		new ReduceDriver<Text, IntWritable, Text, IntWritable>()
		//设置Reducer
		.withReducer(new MaxTemperatureReducer())
		//设置输入key和List
		.withInput(new Text("1950"),  Arrays.asList(new IntWritable(10), new IntWritable(5)))
		//设置期望输出
		.withOutput(new Text("1950"), new IntWritable(10))
		//运行测试
		.runTest();
	}
}
```

通过MRUnit框架对MapReduce测试比较简单，配合JUnit，创建MapperDriver或ReduceDriver对象，设定需要测试的类，设置输入和期望的输出，通过runTest()来运行测试例。

## 6.4 本地运行测试数据

现在Mapper和Reducer已经能够在受控的输入上进行工作了，下一步是创建作业驱动器程序(job driver)，然后在开发机器上使用测试数据运行。

### 6.4.1 在本地作业运行器上运行作业

使用Tool接口，创建MapReduce作业驱动器，寻找最高气温。

**范例6-10 查找最高气温**

``` java
public class MaxTemperatureDriver extends Configured implements Tool {

  @Override
  public int run(String[] args) throws Exception {
    if (args.length != 2) {
      System.err.printf("Usage: %s [generic options] <input> <output>\n",
          getClass().getSimpleName());
      ToolRunner.printGenericCommandUsage(System.err);
      return -1;
    }
    
    Job job = new Job(getConf(), "Max temperature");
    job.setJarByClass(getClass());

    FileInputFormat.addInputPath(job, new Path(args[0]));
    FileOutputFormat.setOutputPath(job, new Path(args[1]));
    
    job.setMapperClass(MaxTemperatureMapper.class);
    job.setCombinerClass(MaxTemperatureReducer.class);
    job.setReducerClass(MaxTemperatureReducer.class);

    job.setOutputKeyClass(Text.class);
    job.setOutputValueClass(IntWritable.class);
    
    return job.waitForCompletion(true) ? 0 : 1;
  }
  
  public static void main(String[] args) throws Exception {
    int exitCode = ToolRunner.run(new MaxTemperatureDriver(), args);
    System.exit(exitCode);
  }
}
```

MaxTemperatureDriver实现了Tool接口，因此能够设置GenericOptionParser支持的选项。现在可以在一些本地文件上运行这个应用。Hadoop有个本地作业运行器，是在MapReduce执行引擎运行单个JVM上的MapReduce作业简化版本，为测试设计。

如果mapreduce.framework.name被设置为local，则使用本地作业运行器。

### 6.4.2 测试驱动程序

除了灵活的配置选项，还可以插入任意Configuration来增加可测试性，来编写测试程序，利用本地作业运行器在已知的输入数据上运行作业，借此来检查输出是否满足预期。

有两种方法可以实现：

- 使用本地作业运行器

	```java
	@Test
	  public void test() throws Exception {
	    Configuration conf = new Configuration();
	    conf.set("fs.defaultFS", "file:///");
	    conf.set("mapreduce.framework.name", "local");
	    conf.setInt("mapreduce.task.io.sort.mb", 1);
	    
	    Path input = new Path("input/ncdc/micro");
	    Path output = new Path("output");
	    
	    FileSystem fs = FileSystem.getLocal(conf);
	    fs.delete(output, true); // delete old output
	    
	    MaxTemperatureDriver driver = new MaxTemperatureDriver();
	    driver.setConf(conf);
	    
	    int exitCode = driver.run(new String[] {
	        input.toString(), output.toString() });
	    assertThat(exitCode, is(0));
	    
	    checkOutput(conf, output);
	  }
	```

	configuration设置了f s.defaultFs和mapreduce.framework.name使用本地文件系统和本地作业运行器。

- 使用mini集群运行。Hadoop有一组测试类MiniDFSCluster、MiniMRCluster和MiniYARNCluster，它以程序的方式创建正在运行的集群，不同于本地作业运行器，它允许在整个HDFS、MapReduce和YARN机器上运行测试。Hadoop的ClusterMapReduceTestCase抽象类提供了一个编写mini集群测试的基础，setUp()和tearDown( )方法可以处理启动和停止时间中的HDFS和YARN集群细节。

## 6.5 在集群上运行

目前，程序已经可以在少量测试数据上正确运行，下面可以准备在Hadoop集群的完整数据集上运行了。

### 6.5.1 打包作业

