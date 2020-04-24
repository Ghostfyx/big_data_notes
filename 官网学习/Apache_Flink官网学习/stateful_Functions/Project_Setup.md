# Project Setup

## 1. Dependency

通过将statefun-sdk添加到现有项目或使用提供的maven原型，可以快速开始构建Stateful Functions应用程序。

```xml
<dependency>
    <groupId>org.apache.flink</groupId>
    <artifactId>statefun-sdk</artifactId>
    <version>2.0.0</version>
</dependency>
```

## 2. Maven Archetype

```
$ mvn archetype:generate \
    -DarchetypeGroupId=org.apache.flink \
    -DarchetypeArtifactId=statefun-quickstart \
    -DarchetypeVersion=2.0.0
```

可以命名新创建的项目。 它将以交互方式要求提供groupId，artifactId和package name。 将有一个与artifact id相同名称的新目录。

```
$ tree statefun-quickstart/
  statefun-quickstart/
  ├── Dockerfile
  ├── pom.xml
  └── src
      └── main
          ├── java
          │   └── org
          │       └── apache
          |            └── flink
          │             └── statefun
          │              └── Module.java
          └── resources
              └── META-INF
                └── services
                  └── org.apache.flink.statefun.sdk.spi.StatefulFunctionModule
```

项目包含四个文件：

- pom.xml
- Module
- org.apache.flink.statefun.sdk.spi.StatefulFunctionModule 用于运行时查找模块的服务条目
- Dockerfile

建议将该项目导入到IDE中进行开发和测试。 

## 3. Build Project

如果要构建/打包项目，请转至项目目录并运行mvn clean package命令。 将找到一个JAR文件，其中包含应用程序，以及可能作为依赖项添加到应用程序的任何库：`target/<artifact-id>-<version>.jar`。