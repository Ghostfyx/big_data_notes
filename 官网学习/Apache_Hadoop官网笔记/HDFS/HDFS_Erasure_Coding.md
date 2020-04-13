# HDFS Erasure Coding

 ## 1. 宗旨

复制非常昂贵。HDFS中默认的3x复制方案在存储空间和其他资源(例如网络带宽)上有200％的开销。但是，对于具有相对较低I/O活动的冷热数据集，在正常操作期间很少访问其他块副本，但是仍然消耗与第一个副本相同的资源量。

因此，一个自然的改进是使用纠删码(EC)代替复制。它提供了相同级别的容错能力，而存储空间却少得多。 在典型的纠删码(EC)设置中，存储开销不超过50％。 EC文件的复制因子没有意义。 它始终为1，无法通过-setrep命令进行更改。

## 2. 背景

在存储系统中，EC最显着的用途是廉价磁盘冗余阵列(RAID)。RAID通过条带化实现EC，条带化将逻辑上连续的数据(例如文件)划分为较小的单位(例如位，字节或块)，并将连续的单位存储在不同的磁盘上。 在本指南的其余部分中，这种条带分布的单位称为条带化单元(或单元)。对于原始数据单元的每个条带，都会计算并存储一定数量的奇偶校验单元，这个过程称为编码。 可以通过基于剩余数据和奇偶校验单元的解码计算来恢复任何条带化单元上的错误。

将EC与HDFS集成可以提高存储效率，同时仍提供与传统的基于复制的HDFS部署类似的数据持久性。 例如，一个具有6个块的3x复制文件将消耗$6\times3 = 18$个磁盘空间。 但是，通过EC(6数据，3奇偶校验)部署，它将仅消耗9块磁盘空间。

## 3. 体系结构

在EC的背景下，条带化具有几个关键优势：

- 启用在线EC(立即以EC格式写入数据)，避免了转换阶段并立即节省了存储空间
- 通过并行利用多个磁盘主轴来增强顺序I/O性能
- 自然地将一个小文件分发到多个DataNode，并且不需要将多个文件捆绑到一个编码组中。 这极大地简化了文件操作，例如删除，配额报告以及联合命名空间之间的迁移

在典型的HDFS群集中，小文件可占总存储消耗的3/4以上。为了更好地支持小文件，HDFS在工作的第一阶段支持带条化的EC。将来，HDFS也将支持连续的EC布局。

- NameNode扩展   条带化HDFS文件在逻辑上由块组组成，每个块组包含一定数量的内部块。 为了减少这些附加块的NameNode内存消耗，引入了新的分层块命名协议。 可以从其任何内部块的ID推断出块组的ID。 这允许在块组而不是块的级别进行管理。

- 客户端扩展  客户端读取和写入路径已得到增强，可以并行处理块组中的多个内部块。在输出/写入路径上，DFSStripedOutputStream管理着一组数据流，每个数据节点在当前块组中存储一个内部块。 数据流异步工作， 协调器负责整个块组的操作，包括结束当前块组，分配新的块组，等等。 在输入/读取路径上，DFSStripedInputStream将请求的逻辑字节数据范围转换为存储在DataNodes上的内部块。 然后，它并行发出读取请求。 发生故障时，它将发出其他读取请求以进行解码。

- DataNode扩展  DataNode运行附加的ErasureCodingWorker（ECWorker）任务，以对失败的纠删码块进行后台恢复。 NameNode检测到失败的EC块，然后NameNode选择一个DataNode进行恢复工作。 恢复任务作为心跳响应传递。 此过程类似于失败时如何重新复制失效的块。 重建执行三个关键任务：

	1. 从源节点读取数据：使用专用线程池从源节点并行读取输入数据。 基于EC策略，它计划对所有源目标的读取请求，并仅读取最少数量的输入块以进行重建。
	2. 解码数据并生成输出数据：从输入数据解码新数据和奇偶校验块。 所有丢失的数据和奇偶校验块一起解码。
	3. 将生成的数据块传输到目标节点：解码完成后，恢复的块将传输到目标DataNode。

- 纠删码策略  为了适应异构的工作负载，我们允许HDFS群集中的文件和目录具有不同的复制和纠删码策略。 纠删码策略封装了如何对文件进行编码/解码。 每个策略由以下信息定义：

	1. EC模式：包括EC组(例如6 + 3)中的数据和奇偶校验块的数量，以及编解码器算法(例如Reed-Solomon，XOR)
	2. 条带化单元的大小： 决定了条带读取和写入的粒度、缓冲区大小和编码工作。

	策略被命名为编解码器-数据块-奇偶校验-块单元大小。当前，支持五种内置策略：RS-3-2-1024k，RS-6-3-1024k，RS-10-4-1024k，RS-LEGACY-6-3-1024k，XOR-2-1- 1024k。

	还支持默认的REPLICATION策略。它只能在目录上设置，以强制目录采用3倍复制方案，而不继承其祖先的纠删码策略。此策略可以使3x复制方案目录与纠删码目录混合使用。
	
	与HDFS存储策略相似，纠删码策略在目录上设置，新建文件时，它将继承其最近祖先目录的EC策略。
	
	目录级别的EC策略仅影响目录中创建的新文件，创建文件后，可以查询其纠删码策略，但不能更改。如果将纠删码文件重命名为具有其他EC策略的目录，则该文件将保留其现有的EC策略。将文件转换为其他EC策略需要重写其数据。 通过复制文件（例如，通过distcp）而不是重命名来做到这一点。
	
	HDFS允许用户通过XML文件自定义EC策略，文件必须包含以下三个部分：
	
	- layoutversion：表示EC策略XML文件格式的版本
	- schema：表示用户定义EC格式
	- policies：包括所有用户定义的EC策略，每个策略都由架构ID和条带化单元的大小(cellsize)组成
	
	Hadoop conf目录中有一个名为user_ec_policies.xml.template的示例EC策略XML文件，用户可以参考该文件。
	
- **Intel ISA-L**：Intel ISA-L代表英特尔智能存储加速库。ISA-L是针对存储应用程序而优化的低级功能的开源集合。 它包括针对Intel AVX和AVX2指令集优化的快速块Reed-Solomon类型擦除代码。 HDFS纠删码可以利用ISA-L加速编码和解码计算。 ISA-L支持大多数主要操作系统，包括Linux和Windows。 默认情况下不启用ISA-L。 请参阅以下说明以了解如何启用ISA-L。

## 4. 部署

### 4.1 集群与硬件设置

EC对集群在CPU和网络方面提出了要求。

编码和解码工作会消耗HDFS客户端和DataNode上的额外CPU。

纠删码要求群集中的DataNode数量最少与配置的EC条带宽度一样。对于EC策略(6,3)，则需要至少9个DataNode。

纠删码文件也分布在整个机架中，以实现机架容错。 这意味着在读写条带化文件时，大多数操作都是在机架上进行的。 因此，网络对分带宽非常重要。

 对于机架容错，拥有足够数量的机架也很重要，因此平均而言，每个机架所容纳的块数不超过EC奇偶校验块的数。 计算此公式的公式将是(数据块+奇偶校验块)/奇偶校验块，并取整。这意味着最少3个机架(按$(6+3)/3=3$计算），理想情况下为9个或更多，以处理计划内和计划外的停机。 对于机架数少于奇偶校验单元数的群集，HDFS无法维持机架容错能力，但仍将尝试在多个节点之间分布条带化文件以保留节点级容错能力。 因此，建议设置具有类似数量的DataNode的机架。

### 4.2 配置关键

默认情况下，所有内置纠删码策略均被禁用，但dfs.namenode.ec.system.default.policy中定义的默认除外。 群集管理员可以根据群集的大小和所需的容错属性，通过`hdfs ec [-enablePolicy -policy <policyName>]`命令启用策略集。 例如，对于具有9个机架的群集，像RS-10-4-1024k这样的策略将不会保留机架级的容错能力，而RS-6-3-1024k或RS-3-2-1024k可能更合适 。 如果管理员仅关心节点级的容错能力，则只要集群中至少有14个DataNode，RS-10-4-1024k仍然适用。

可以通过`dfs.namenode.ec.system.default.policy`配置来配置系统默认EC策略。 使用此配置时，如果在`-setPolicy`命令中未将任何策略名称作为参数传递，则将使用默认的EC策略。

默认情况下，`dfs.namenode.ec.system.default.policy`为`RS-6-3-1024k`。

可以使用以下客户端和DataNode配置密钥来配置Reed-Solomon和XOR的编解码器实现：用于默认RS编解码器的`io.erasurecode.codec.rs.rawcoders`，用于XOR编解码器的`io.erasurecode.codec.xor.rawcoders`。

用户还可以使用配置密钥来配置自定义编解码器，例如：`io.erasurecode.codec.self-defined-codec.rawcoders`。这些键的值是带有备用机制的编码器名称列表。这些编解码器工厂将按照配置值指定的顺序加载，直到成功加载编解码器为止。默认的RS和XOR编解码器配置比纯Java首选本地实现。没有RS-LEGACY本机编解码器实现，因此默认设置仅为纯Java实现。所有这些编解码器均具有纯Java实现。对于默认的RS编解码器，还有一个本机实现，可利用英特尔ISA-L库来提高编解码器的性能。对于XOR编解码器，还支持利用英特尔ISA-L库提高编解码器性能的本机实现。请参阅“Enable Intel ISA-L”一节以获取更多详细信息。 RS Legacy的默认实现是纯Java，默认RS和XOR的默认实现是使用Intel ISA-L库的本机实现。

还可以通过以下配置参数来调整DataNode上的EC后台恢复工作：

- `dfs.datanode.ec.reconstruction.stripedread.timeout.millis`-条带读取的超时。默认值为5000毫秒
- `dfs.datanode.ec.reconstruction.stripedread.buffer.size`-读取器服务的缓冲区大小。默认值为64KB
- `dfs.datanode.ec.reconstruction.threads-Datanode`用于后台重建工作的线程数。默认值为8个线程。
- `dfs.datanode.ec.reconstruction.xmits.weight`-EC后台恢复任务与复制块恢复相比使用的**xmit**的相对权重。默认值为0.5。设置为0以禁用EC恢复任务的计算权重，即EC任务始终具有1xmits。纠删码恢复任务的xmit被计算为读取流的数量和写入流的数量之间的最大值。例如，如果EC恢复任务需要从6个节点读取并写入2个节点，则其xmit的最大值为(6,2)* 0.5 =3。复制文件的恢复任务始终计为1xmit。 NameNode利用`dfs.namenode.replication.max-streams`减去DataNode上的总xmitsInProgress(将来自复制文件和EC文件的xmit组合在一起)来安排恢复任务到此DataNode。

### 4.3 启用Intel ISA-L

默认RS编解码器的HDFS本机实现利用Intel ISA-L库来改善编码和解码计算。 要启用和使用Intel ISA-L，需要执行三个步骤：

1. 建立ISA-L库。 请参阅官方网站“ https://github.com/01org/isa-l/”以获取详细信息
2. 使用ISA-L支持构建Hadoop。 请参阅源代码（BUILDING.txt）中“ Hadoop的构建说明”中的“ Intel ISA-L构建选项”部分
3. 使用-Dbundle.isal将isal.lib目录的内容复制到最终的tar文件中。 使用tar文件部署Hadoop。 确保ISA-L在HDFS客户端和DataNode上可用

要验证Hadoop是否正确检测到ISA-L，请运行hadoop checknative命令。

### 4.4 管理员命令

HDFS提供了一个EC子命令来执行与纠删码有关的管理命令。

```
   hdfs ec [generic options]
     [-setPolicy -path <path> [-policy <policyName>] [-replicate]]
     [-getPolicy -path <path>]
     [-unsetPolicy -path <path>]
     [-listPolicies]
     [-addPolicies -policyFile <file>]
     [-listCodecs]
     [-enablePolicy -policy <policyName>]
     [-disablePolicy -policy <policyName>]
     [-help [cmd ...]]
```

- [-setPolicy -path <path> [-policy <policyName>] [-replicate]]

	-replicate 在目录上应用默认的REPLICATION方案，强制目录采用3x复制方案。

	-replicate和-policy <policyName>是可选参数。 不能同时指定它们。

-  [-addPolicies -policyFile < file>]

	添加用户定义的纠删码策略列表。请参阅etc/hadoop/ user_ec_policies.xml.template获取示例策略文件。 单元格的最大大小在属性“ dfs.namenode.ec.policies.max.cellsize”中定义，默认值为4MB。 当前，HDFS允许用户总共添加64个策略，并且添加的策略ID在64到127之间。如果已经添加了64个策略，添加策略将失败。

- [-listCodecs]

	获取系统中支持的纠删码编解码器和编码器的列表。 编码器是编解码器的一种实现。 编解码器可以具有不同的实现，因此可以具有不同的编码器。 编解码器的编码器以后备顺序列出。