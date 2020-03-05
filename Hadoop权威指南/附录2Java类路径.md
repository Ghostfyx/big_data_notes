# JAVA 类路径

Java类路径告诉 *java* 解释器和 *javac* 编译器去哪里找它们要执行或导入的类。类（*.class* 文件）可以存储在目录或 *jar* 文件中，或者存储在两者的组合中，但是只有在它们位于类路径中的某个地方时，*Java* 编译器或解释器才可以找到它们。

在 *Windows* 中，类路径中的多个项是用分号分隔（ *;*）的，而在 *UNIX* 中，这些项是用冒号分隔（ *:*）的。

## *Java*程序的启动命令详解

java程序的启动方式为：

```
java -cp 类路径 全限定的类名 参数1 参数2 参数3
```

### *Java*命令

在windows上，它是没有显示地写上exe后缀的可执行程序。大家都知道在计算机中，要指明一个文件，仅文件名是不够的，而是需要完整的路径才能唯一的定位它。但此处为什么可以只写一个程序名字？只是因为java安装目录下面的bin目录被加到了系统的path环境变量中，而这个添加操作通常是在安装JRE时自动完成的。如果bin目录没有被添加到path环境变量中（这种情况可能出现在随便的手动拷贝jre目录的时候），则此处就要写完整了，例如：C:\Program Files (x86)\Java\jre1.8.0_121\bin\java -cp ……

### -cp

-cp选项用于罗列类路径的位置，它是-classpath的缩写。类路径的作用是告诉java虚拟机从哪些地方查找类。类路径的几个要点：

- 类路径中可以指定三种位置：文件夹、jar文件、zip文件；
- cp或者-classpath可以指定多个位置，在windows上是用分号隔开，在linux上是用冒号隔开。例如在linux上：-cp dir1:dir2:dir3，此处指定了3个目录作为类查找路径。
- 如果没有明确的指定类路径，则默认是当前工作路径，注意当前工作路径是一个文件夹，因此如果当前工作路径下面有个jar文件，jar文件中的类是不会被找到的。**记住文件夹与jar各是各**。
- 如果通过-cp或者-classpath选项指定了类路径，则当前工作路径就不会再包含进类路径中了。此时如果仍然需要将当前工作路径纳入类路径，需要通过点号再加回来。例如：-cp .:dir1，此处表示在linux上当前工作目录和dir1目录都是类路径；
- 路径通配符的使用，通配符是为了减少了在指定类路径时罗列jar麻烦。有以下几个注意点：
	1. 通配符只是用来匹配jar的，不能用来匹配类文件或者目录。
	2. 通配符不递归匹配。
	3. 如果目录dir下面有6个jar都要作为类路径，传统的可以写成：-cp test1.jar:test2.jar:test3.jar:test4.jar:test5.jar:test6.jar，有没有发现很麻烦，其实用通配符写起来简单多了：-cp *，此时如果当前目录也要作为类路径，可写成：-cp .:*。
- 还可以通过CLASSPATH环境变量指定类路径，但是**不到万不得已，不要这样用**。如果配置到环境变量中去了，则系统中所有的java程序都会相互影响。

## 设置 *Java* 类路径

有三种方式设置 *Java* 类路径：

1．永久地，通过在系统级上设置 *CLASSPATH* 环境变量来实现。使用控制面板的系统设置来添加名为 *CLASSPATH* 的新变量，从而永久性地设置 *Windows* 环境变量。UNIX 用户可以通过向 *.profile* 或 *.cshrc* 文件添加 *CLASSPATH* 变量来永久设置类路径。

2．临时地，通过在命令窗口或 *shell* 中设置 *CLASSPATH* 环境变量来实现。在 *Windows* 命令窗口中临时设置 CLASSPATH

```
C:/>set CLASSPATH=%CLOUDSCAPE_INSTALL%/lib/cs.jar;
```

如果是临时设置类路径，那么每次打开新的命令窗口时，都需要再次设置它。

3．在运行时进行，每次启动 *Java* 应用程序和 *JVM*，都要指定类路径。运行时使用 *-cp* 选项来指定类路径，这里的运行时是指启动应用程序和 *JVM* 时。

##*Java* ClassPath示例

下面是在IDEA启动Java一个类，控制台打印的类路径信息，可以发现：

- 加载了配置在系统ClassPath下的jre/lib下的所有jar包；
- 加载了运行主类的类路径与全限定类名

```sh
-classpath /Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/jre/lib/charsets.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/jre/lib/deploy.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/jre/lib/ext/cldrdata.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/jre/lib/ext/dnsns.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/jre/lib/ext/jaccess.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/jre/lib/ext/jfxrt.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/jre/lib/ext/localedata.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/jre/lib/ext/nashorn.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/jre/lib/ext/sunec.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/jre/lib/ext/sunjce_provider.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/jre/lib/ext/sunpkcs11.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/jre/lib/ext/zipfs.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/jre/lib/javaws.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/jre/lib/jce.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/jre/lib/jfr.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/jre/lib/jfxswt.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/jre/lib/jsse.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/jre/lib/management-agent.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/jre/lib/plugin.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/jre/lib/resources.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/jre/lib/rt.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/lib/ant-javafx.jar:/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/lib/dt.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/lib/javafx-mx.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/lib/jconsole.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/lib/packager.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/lib/sa-jdi.jar:
/Library/Java/JavaVirtualMachines/jdk1.8.0_202.jdk/Contents/Home/lib/tools.jar:
/Users/yuexiangfan/coding/JavaProject/Leet_Code_Practise/target/classes string.week_4.RomanToInt_13
```

# *Java*项目的ClassPath

