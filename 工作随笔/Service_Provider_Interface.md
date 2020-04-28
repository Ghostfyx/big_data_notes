# SPI(Service Provider Interface)简介

## 1. SPI简介

SPI全称为(Service Provider Interface)，是JDK内置的一种服务提供发现机制。

一个服务(Service)通常指的是已知的接口或者抽象类，服务提供方就是对这个接口或者抽象类的实现，然后按照SPI 标准存放到资源路径META-INF/services目录下，文件的命名为该服务接口的全限定名。如有一个服务接口：

```java
public interface DemoService {
    public String sayHi(String msg);
}
```

服务实现类为：

```java
public class DemoServiceImpl implements DemoService {

    @Override
    public String sayHi(String msg) {

        return "Hello, "+msg;
    }

}
```

那此时需要在META-INF/services中创建一个名为com.ricky.codelab.spi.DemoService的文件，其中的内容就为该实现类的全限定名：com.ricky.codelab.spi.impl.DemoServiceImpl。

如果该Service有多个服务实现，则每一行写一个服务实现（#后面的内容为注释），并且该文件只能够是以UTF-8编码。

然后，可以通过`ServiceLoader.load(Class class)`，来**动态加载**Service的实现类了。

许多开发框架都使用了Java的SPI机制，如java.sql.Driver的SPI实现（mysql驱动、oracle驱动等）、common-logging的日志接口实现、dubbo的扩展实现，Hadoop的客户端协议提供者的实现(ProtocolProvider)等等。

## 2. SPI机制的约定

- 在META-INF/services/目录中创建以Service接口全限定名命名的文件，该文件内容为Service接口具体实现类的全限定名，文件编码必须为UTF-8
- 使用ServiceLoader.load(Class class)，动态加载Service接口的实现类
- 如果SPI的实现类为jar，则需要将其放在当前程序的classpath下
- Service的具体实现类必须有一个不带参数的构造方法
	