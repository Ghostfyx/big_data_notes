# 服务调用RPC&Restful深入理解

## 1. RPC

RPC 即远程过程调用(Remote Procedure Call Protocol，简称RPC)，**像调用本地服务方法一样调用服务器的服务方法**。通常的实现有 **XML-RPC , JSON-RPC** , 通信方式基本相同, 所不同的只是传输数据的格式。

RPC是**分布式架构的核心**，按**响应方式**分如下两种：

- 同步调用：客户端调用服务方方法，等待直到服务方返回结果或者超时，再继续自己的操作
- 异步调用：客户端把消息发送给中间件，不再等待服务端返回，直接继续自己的操作

同步调用的实现方式有WebService和RMI。Web Service提供的服务是基于web容器的，底层使用http协议，因而适合不同语言异构系统间的调用。RMI实际上是Java语言的RPC实现，允许方法返回 Java 对象以及基本数据类型，适合用于JAVA语言构建的不同系统间的调用。


异步调用的JAVA实现版就是JMS(Java Message Service)，目前开源的的JMS中间件有Apache社区的ActiveMQ、Kafka消息中间件，另外有阿里的RocketMQ。

RPC架构里包含如下4个组件：

- 客户端(Client)：服务调用方
- 客户端存根(Client Stub)：存放服务端地址信息，将客户端的请求参数打包成网络消息，再通过网络发送给服务方
- 服务端存根(Server Stub)：接受客户端发送过来的消息并解包，再调用本地服务
- 服务端：服务提供者

![](./img/rpc.jpg)

具体实现步骤：

-  服务调用方(client)客户端以本地调用方式调用服务；
- client stub接收到调用后负责将方法、参数等组装成能够进行网络传输的消息体；在Java里就是序列化的过程
- client stub找到服务地址，并将消息通过网络发送到服务端；
- server stub收到消息后进行解码,在Java里就是反序列化的过程；
-  server stub根据解码结果调用本地的服务；
-  本地服务执行处理逻辑；
-  本地服务将结果返回给server stub；
-  server stub将返回结果打包成消息，Java里的序列化；
- server stub将打包后的消息通过网络并发送至消费方
- client stub接收到消息，并进行解码, Java里的反序列化
- 服务调用方（client）得到最终结果

RPC框架的目标就是把2-10步封装起来，把调用、编码/解码的过程封装起来，让用户像调用本地服务一样的调用远程服务。要做到对客户端(调用方)透明化服务， RPC框架需要考虑解决如下问题： 

1. **通讯问题** : 主要是通过在客户端和服务器之间建立TCP连接，远程过程调用的所有交换的数据都在这个连接里传输。连接可以是按需连接，调用结束后就断掉，也可以是长连接，多个远程过程调用共享同一个连接。
2.  **寻址问题**：A服务器上的应用怎么告诉底层的RPC框架，如何连接到B服务器(如主机或IP地址)以及特定的端口，方法的名称是什么，这样才能完成调用。比如基于Web服务协议栈的RPC，就要提供一个endpoint URI，或者是从UDDI服务上查找。如果是RMI调用的话，还需要一个RMI Registry来注册服务的地址。
3. **序列化与反序列化** ：当A服务器上的应用发起远程过程调用时，方法的参数需要通过底层的网络协议如TCP传递到B服务器，由于网络协议是基于二进制的，内存中的参数的值要序列化成二进制的形式，也就是序列化（Serialize）或编组（marshal），通过寻址和传输将序列化的二进制发送给B服务器。 同理，B服务器接收参数要将参数反序列化。B服务器应用调用自己的方法处理后返回的结果也要序列化给A服务器，A服务器接收也要经过反序列化的过程。

## 2. REST

REST即表述性状态传递（Representational State Transfer，简称REST），是一种软件架构风格。REST通过HTTP协议定义的通用动词方法(GET、PUT、DELETE、POST) ，以URI对网络资源进行唯一标识，响应端根据请求端的不同需求，通过无状态通信，对其请求的资源进行表述。 

Rest架构的主要原则：

1. 网络上的所有事物都被抽象为资源
2. 每个资源都有一个唯一的资源标识符
3. 同一个资源具有多种表现形式(xml,json等)
4. 对资源的各种操作不会改变资源标识符
5. 所有的操作都是无状态的

REST使用HTTP+URI+XML /JSON 的技术来实现其API要求的架构风格：HTTP协议和URI用于统一接口和定位资源，文本、二进制流、XML、JSON等格式用来作为资源的表述。

RESTful风格的体现，在你使用了get请求，就是查询；使用post请求,就是新增的请求；使用put请求，就是修改的请求；使用delete请求，就是删除的请求。这样做就完全没有必要对crud做具体的描述。

满足REST约束条件和原则的架构，就被称为是RESTful架构。就像URL都是URI(统一资源标识)的表现形式一样，RESTful是符合REST原则的表现形式。

### 2.1 Sparing实现restful服务

SpringMVC原生态的支持了REST风格的架构设计

所涉及到的注解：

- @RequestMapping
- @PathVariable
- @ResponseBody

```
@RequestMapping(value = "{id}", method = RequestMethod.GET)  
@ResponseBody  

public enum RequestMethod {

	GET, HEAD, POST, PUT, PATCH, DELETE, OPTIONS, TRACE

}
```

以Apache Thrift为代表的二进制RPC，支持多种语言（但不是所有语言），四层通讯协议，性能高，节省带宽。相对Restful协议，使用Thrifpt RPC，在同等硬件条件下，带宽使用率仅为前者的20%，性能却提升一个数量级。但是这种协议最大的问题在于，无法穿透防火墙。

以Spring Cloud为代表所支持的Restful协议，优势在于能够穿透防火墙，使用方便，语言无关，基本上可以使用各种开发语言实现的系统，都可以接受Restful的请求。但性能和带宽占用上有劣势。

所以，业内对微服务的实现，基本是确定一个组织边界，在该边界内，使用RPC; 边界外，使用Restful。这个边界，可以是业务、部门，甚至是全公司。
