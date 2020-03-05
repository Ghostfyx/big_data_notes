# Hadoop ClassPath

编写实际生产用的hadoop mapreduce程序的时候，通常都会引用第三方库，也就会碰到ClassPath的问题，主要是两种情况：

- 找不到类ClassNotFound
- 库的加载顺序不对，就是第三库引用了一个比较通用的库，比如jackson-core-asl，而hadoop包含了这个库，但是版本稍低，默认情况下hadoop会优先加载自身包含的库，这样就会造成引用库的版本不对，从而出现找不到类会类中的方法的错误

一般会在两个阶段碰到，分别有不同的解决方法：

- 作业提交阶段
- task运行阶段

## 作业提交阶段

找不到类这种情况有两种解决方法：

- 将第三方库和自己的程序代码打包到一个jar包中
- 设置HADOOP_CLASSPATH环境变量

### 打包

使用maven打包，pom文件中打包插件设置如下：

```xml
<build>
        <plugins>
            <plugin>
                <artifactId>maven-assembly-plugin</artifactId>
                <configuration>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                </configuration>
                <executions>
                    <execution>
                        <id>make-assembly</id>
                        <phase>package</phase>
                        <goals>
                            <goal>single</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>

```

### 设置环境变量

假如要将jackson-core-asl-1.9.13.jar库添加到HADOOP_CLASSPATH中：

```sh
export HADOOP_CLASSPATH=jackson-core-asl-1.9.13.jar
```


库加载顺序不对这种情况，那就是要让hadoop优先加载用户指定的库，设置如下的环境变量：

```sh
export HADOOP_USER_CLASSPATH_FIRST=true
```

这样设置之后可以解决问题的原因是，hadoop jar命令启动的程序的时候，会通过hadoop-config.sh来设置classpath，其中有段这样的设置代码：

```sh
if [[ ( "$HADOOP_CLASSPATH" != "" ) && ( "$HADOOP_USE_CLIENT_CLASSLOADER" = "" ) ]]; then
  # Prefix it if its to be preceded
  if [ "$HADOOP_USER_CLASSPATH_FIRST" != "" ]; then
    CLASSPATH=${HADOOP_CLASSPATH}:${CLASSPATH}
  else
    CLASSPATH=${CLASSPATH}:${HADOOP_CLASSPATH}
  fi
fi
```

也就是说如果HADOOP_CLASSPATH不为空且HADOOP_USER_CLASSPATH_FIRST不为空的时候，会将HADOOP_CLASSPATH指定的类库优先加载。

## Task运行阶段

找不到类这种情况，通过-libjars指定引用的库，hadoop将这里指定的库上传到DistributedCache中，然后添加到task运行的classpath。

库加载顺序的问题，可以通过设置Configuration类解决。

```java
conf.set("mapreduce.job.user.classpath.first","true");
conf.set("mapreduce.task.classpath.user.precedence","true");
```

