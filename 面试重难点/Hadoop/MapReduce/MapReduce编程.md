# MapReduce编程

## 1. MapReduce 编程规范

用户编写的程序分成三个部分：Mapper、Reducer 和 Driver。

### 1.1 Map阶段

- 用户自定义的Mapper要继承Map类
- Mapper的输入数据是KV对的形式(KV的类型可自定义)
- Mapper中的业务逻辑写在map()方法中
- Mapper的输出数据是KV对的形式(KV的类型可自定义)
- map()方法(MapTask进程)对每一个调用一次

### 1.2 Reduce阶段

- 用户自定义的Reducer要继承Reduce类
- Reducer的输入数据类型对应Mapper的输出数据类型
- Reducer的业务逻辑写在reduce()方法中
- ReduceTask进程对**每一组相同k**的组调用一次reduce()方法

### 1.3 Driver阶段

相当于YARN集群的客户端，用于提交整个程序到YARN集群，提交的是 封装了MapReduce程序相关运行参数的job对象。