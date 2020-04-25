# Hadoop序列化

## 1. 序列化概述

### 1.1 什么是序列化

序列化就是把内存中的对象，转换成字节序列(或其他数据传输协议)以便于存储到磁盘(持久化)和网络传输。 

反序列化就是将收到字节序列(或其他数据传输协议)或者是磁盘的持久化数据，转换成内存中的对象。

### 1.2 序列化原因

一般来说，“活的”对象只生存在内存里，关机断电就没有了。而且“活的” 对象只能由本地的进程使用，不能被发送到网络上的另外一台计算机。 然而序 列化可以存储“活的”对象，可以将“活的”对象发送到远程计算机。

### 1.3 Java序列化

Java的序列化是一个重量级序列化框架(Serializable)，一个对象被序列化后，会 附带很多额外的信息(各种校验信息，Header，继承体系等)，不便于在网络中高效 传输。

### 1.4 Hadoop序列化

Hadoop开发了一套序列化机制(Writable)。有以下几个特点：

- 紧凑 ：高效使用存储空间
- 快速：读写数据的额外开销小
- 可扩展：随着通信协议的升级而可升级
- 可扩展：随着通信协议的升级而可升级

自定义Bean对象实现序列化，需要实现Writable接口：

- 有空参构造

- 重写序列化方法

	```java
	public void write(DataOutput out) throws IOException {}
	```

- 重写反序列化方法

	```java
	public void readFields(DataInput in) throws IOException 
	```

- 注意反序列化的顺序和序列化的Bean字段顺序完全一致

- 要想把结果显示在文件中，需要重写 toString()方法

- 如果需要将自定义的 bean 放在 key 中传输，则还需要实现Comparable 接口，因为 MapReduce 框中的 Shuffle 过程要求对 key 必须能排序

	```java
	public int compareTo(FlowBean o) 
	```

	

