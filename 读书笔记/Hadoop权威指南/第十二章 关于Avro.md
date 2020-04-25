# 第十二章 关于Avro

Apache Avro是一个独立于编程语言的数据序列化系统。该项目由Doug Cutting(Hadoop之父)创建，旨在解决Hadoop中Writable类型的不足：缺乏语言可移植性。拥有一个可被多种语言(当前是C 、C++、C#、Java、PHP、Python和Ruby)处理的数据格式与绑定到单一语言的数据格式相比，前者更给予与公共共享数据集。Avro同时也更具有生命力，该语言将使得数据具有更长的生命周期，即使原先用于读/写数据的语言已经不再使用。

但是为什么要有一个新的数据序列化系统？与Apache Thrift和Google的Protocol Buffers相比，Avro有其独有的特性。与前述系统及其他系统相似，Avro数据是用语言无关的模型定义的。但是与其他系统不同的是，在Avro中，代码生成是可选的，这意味着你可以对遵循制定模式的数据进行读/写操作，即使在此之前代码从来没有见过这个特殊的数据模式。为此，Avro假设数据模式总是存在的(在读/写数据时)，它形成的时非常精简的编码，因为编码后的数据不需要用字段标识符来打标签。

Avro模式通常使用JSON来写，数据通常采用二进制格式编码，但也有其他选择。还有一种高级语言称为Avro IDL。可以使开发人员更为熟悉的类C语言来写模式。还有一个基于JSON的数据编码方式(对构建原型和调试Avro数据很有用，因为它们时人类可读的)。

Avro规范对所有实现都必须支持的二进制格式进行了精确定义，同时还制定了这些实现需要支持的其他Avro特性，但是，该规范并没有为API制定规范：实现可以根据自己的需求操作Avro数据并给出相应的API，因为每个API都与语言相关，重要的二进制格式只有一种这一事实意味着绑定心的编程语言的门槛比较低，可以避免语言和格式组合爆炸的问题，否则对相互操作性造成一定的问题。

Avro具有丰富的**模式解析(schema resolution)**能力。在精心定义的约束下，该数据所用的模式不必与写数据所用的模式相同。由此，Avro是支持模式演化的。例如，如果有一个新的，可选择的字段要加入记录中，那么只需要在用来读取老数据的模式中声明它即可。新客户端和以前的客户端非常相似，均能读取按旧模式存储的数据，同时心的客户端可以使用新字段写入新的内容。相反，如果老客户端读取新客户端写入的数据，会忽略新加入的字段并按照先前的数据模式处理。

Avro为一系列对象指定了一个对象容器，类似于Hadoop的顺序文件。Avro数据文件包含元数据项(模式数据存储在其中)，使此文件可以自我生命。Avro数据文件支持压缩，并且是可切分的，这对MapReduce的输入格式至关重要。对Avro的支持远不止于MapReduce。例如：Pig、Hive、Crunch、Spark都能读/写Avro数据文件。

Avro还可用于RPC。

## 12.1 Avro数据类型和模式

Avro定义了少量的基本数据类型，通过编写模式的方式，它们可被用于构建应用特定的数据结构，考虑到相互操作性，实现必须支持所有的Avro类型。

表12-1列举了Avro的基本类型，每个基本类型也可以使用type属性来指定，其结果是形式更加冗长，示例如下：

```json
{"type":"null"}
```

每个Avro语言的API都包含该语言特定的Avro类型表示，例如：Avro的double类型可以使用C、C++、Java的double类型，Python的float类型以及Ruby的Float类型表示：

​																**表12-1 Avro基本类型**

| 类型    | 描述                       | 模式示例  |
| ------- | -------------------------- | --------- |
| null    | 空值                       | "null"    |
| boolean | 二进制值                   | "boolean" |
| int     | 32位带符号整数             | "Int"     |
| long    | 64位带符号整数             | "long"    |
| float   | 单精度(32位)IEEE 754浮点数 | "float"   |
| double  | 单精度(64位)IEEE 754浮点数 | "double"  |
| bytes   | 8位无符号字节序列          | "bytes"   |
| string  | Unicode字符序列            | "strings" |

表12-2列举了Avro的复杂类型，并为每种类型给出相应的模式示例。

​																**表12-2 Avro复杂类型**

| 类型   | 描述                                                         | 模式示例                                                     |
| ------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| array  | 一个排过序的对象集合，制定数组中所有对象必须模式相同         | {"type":"array",“items”:"long"}                              |
| map    | 未排过序的键-值对，键必须是字符串，值可以是任何一种类型，但是一个特性mao中所有值必须模式相同 | {"type":"map","values":"string"}                             |
| record | 一个任意类型的命名字段集合                                   | {<br/>"type": "record",<br/>"name": "WeatherRecord",<br/>"doc": "A weather reading.",<br/>"fields": [<br/>{"name": "year", "type": "int"},<br/>{"name": "temperature", "type": "int"},<br/>{"name": "stationId", "type": "string"}<br/>]<br/>} |
| enum   | 一个命名的值集合                                             | {<br/>"type": "enum",<br/>"name": "Cutlery",<br/>"doc": "An eating utensil.",<br/>"symbols": ["KNIFE", "FORK", "SPOON"]<br/>}<br/> |
| fixed  | 一组特定数量的8位无符号字节                                  | {<br/>"type": "fixed",<br/>"name": "Md5Hash",<br/>"size": 16<br/>} |
| union  | 模式的并集，并集可用JSON数组表示，其中每个元素为一个模式，并集表示的数据必须与其内的某个模式匹配 | [<br/> "null",<br/>"string",<br/>{"type": "map", "values": "string"}<br/>] |

而且，一种语言可以有多种表示或映射。所有语言都支持动态映射，即使运行前并不知道具体模式，可以是使用动态映射。对此，Java称为 **通用映射(generic mapping)**。

另外，Java和C++实现可以自动生成代码来表示符合某种Avro模式的数据。如果在读/写数据之前就有模式备份的话，代码生成(code generation)能优化数据处理。Java中称为 **特殊映射(specific mapping)**。

Java还有第三类映射，即**自反映射(reflect mapping)**，将Avro类型映射到已有的Java类型，他的速度比上述两种映射都慢，但是Avro能够自动推断模式。

表12-3列举了java的类型映射。如表所示，除非有特别说明，否则特殊映射和通用映射相同，自反映射与特殊映射相同，特殊映射与通用映射仅在reocrd、enum和fixed三个类型上有区别，他们的特殊映射都有自动生成的类，类名由name属性和可选的namespace属性决定。

​													**表12-3 Avro的Java类型映射**

| Avro类型 | Java通用映射                          | Java特殊映射                                            | Java自定义映射                                             |
| -------- | ------------------------------------- | ------------------------------------------------------- | ---------------------------------------------------------- |
| null     | null类型                              |                                                         |                                                            |
| boolean  | boolean                               |                                                         |                                                            |
| int      | int                                   |                                                         | short或int                                                 |
| long     | long                                  |                                                         |                                                            |
| float    | float                                 |                                                         |                                                            |
| double   | double                                |                                                         |                                                            |
| bytes    | java.nio.bytebuffer                   |                                                         | 字节数组                                                   |
| string   | org.apache.avro.util.utf8             |                                                         | java.lang.string                                           |
| array    | org.apache.avro.generic.GenericArray  |                                                         | 数组或java.util.Collection                                 |
| map      | java.util.map                         |                                                         |                                                            |
| record   | org.apache.avro.generic.genericrecord | 生成实现org.apache.avro.specific.specificrecord类的实现 | 具有零参数构造函数的任意用户类，继承了所有不传递的实例字段 |
| enum     | java.lang.string                      | 生成Java enum类型                                       | 任意Java Enum类型                                          |
| fixed    | org.apache.avro.generic.genericfixed  | 生成实现org.apache.avro.specific.specificFixed类的实现  | org.apache.avro.generic.genericfixed                       |
| union    | Java.lang.Object                      |                                                         |                                                            |

