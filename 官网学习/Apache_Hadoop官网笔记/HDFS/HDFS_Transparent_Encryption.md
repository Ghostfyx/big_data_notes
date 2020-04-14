# Transparent Encryption in HDFS

## 1. 概述

HDFS实现了透明的端到端加密。配置完成后，可以透明地加密和解密从特殊HDFS目录读取和写入的数据，而无需更改用户应用程序代码。 这种加密也是端到端的，这意味着数据只能由客户端加密和解密。 HDFS从不存储或访问未加密的数据或未加密的数据加密密钥。 这满足了加密的两个典型要求：静态加密(代表永久存储介质上的数据，例如：磁盘)以及传输中加密(例如，当数据通过网络传输时)。

## 2. 背景

可以在传统数据管理软件/硬件堆栈的不同层上进行加密。 选择在给定层进行加密有不同的优点和缺点：

- **应用层级加密(Application-level encryption)**：最安全，最灵活的方法。 该应用对加密的内容具有最终控制权，并且可以精确反映用户的需求。 但是，编写应用程序来做到这一点很困难。 对于不支持加密的现有应用的客户，这也不是一种选择。
- **数据库层级加密(Database-level encryption)**：就其属性而言，类似于应用程序级加密。 大多数数据库供应商都提供某种形式的加密。 但是，可能存在性能问题。 此外索引不能被加密。
- **文件系统级别加密(Filesystem-level encryption)**：此选项提供高性能，应用程序透明性，并且通常易于部署。 但是，它无法为某些应用程序级加密策略建模。 例如，多租户应用程序可能希望基于最终用户进行加密。 数据库可能希望为单个文件中存储的每个列使用不同的加密设置。
- **磁盘级别加密(Disk-level encryption)**：易于部署和高性能，但也相当不灵活。 只有真正防止物理泄密。

HDFS级别的加密适合上述堆栈中的数据库级别和文件系统级别的加密。它有很多优点： HDFS加密能够提供良好的性能，而现有的Hadoop应用程序能够在加密的数据上透明运行。 在制定策略决策时，HDFS还具有比传统文件系统更多的上下文。

HDFS级别的加密还可以防止在文件系统级别及文件系统以下的攻击，即所谓的OS级别攻击。操作系统和磁盘仅与加密字节进行交互，因为数据已由HDFS加密。

## 2. 用例

许多不同的政府，金融和监管实体都要求数据加密。 例如，医疗保健行业有HIPAA法规，卡支付行业有PCI DSS法规，而美国政府有FISMA法规。 HDFS内置了透明加密，使组织更容易遵守这些法规。

加密也可以在应用程序级别执行。通过将其集成到HDFS中，现有应用程序可以对加密数据进行操作而无需更改程序。 这种集成架构意味着加密文件语义以及与其他HDFS功能的更好协调。

## 3. 体系结构

### 3.1 概述

对于透明加密，介绍HDFS的一个新的抽象概念： **the encryption zone**。加密区域是一个特殊的目录，其内容在写入时将被透明加密，在读取时将被透明解密。 每个加密区域都与创建该区域时指定的单个加密区域密钥相关联。加密区域内的每个文件都有其自己的唯一数据加密密钥(data encryption key，DEK)。 HDFS从不直接处理DEK。 相反，HDFS仅处理加密的数据加密密钥(encrypted data encryption key,EDEK)。客户端解密EDEK，然后使用后续的DEK读取和写入数据。 HDFS数据节点仅看到加密字节流。

加密的一个非常重要的用例是将其打开并确保整个文件系统中的所有文件都已加密。 为了支持这种有力的保障，同时不失去在文件系统的不同部分中使用不同加密区域密钥的灵活性，HDFS允许嵌套加密区域。在一个加密区域创建后(例如：在根目录上)。用户可以使用不同的密钥在其子目录(例如：/home/alice)上创建更多的加密区域。使用最接近的父目录加密区域的密钥生成文件的EDEK。

需要新的群集服务来管理加密密钥：Hadoop密钥管理服务器(KMS)。 在HDFS加密的情况下，KMS履行三项基本职责：

- 提供对存储加密区域密钥的访问
- 生成新的加密数据加密密钥并存储在NameNode上
- 解密供HDFS客户端使用的加密数据加密密钥(注意：是加密数据密钥的加密)

### 3.2 在加密区域内访问数据

当在加密区域内新建文件，NameNode请求KMS生成一个用加密区域的密钥加密的新EDEK。 然后，EDEK作为文件元数据的一部分永久存储在NameNode上。

当在加密区域内读取文件，NameNode为客户端提供文件的EDEK和用于加密EDEK的加密区域密钥版本。随后，客户端请求KMSKMS解密EDEK，其中涉及检查客户端是否有权访问加密区域密钥版本。 假设成功，客户端将使用DEK解密文件的内容。

读写路径的所有上述步骤都是通过DFSClient，NameNode和KMS之间的交互自动发生的。

对加密文件数据和元数据的访问由正常的HDFS文件系统权限控制。 这意味着，如果HDFS受到破坏(例如：通过获得对HDFS超级用户帐户的未授权访问)，则恶意用户只能获得对密文和加密密钥的访问。 但是，由于对加密区域密钥的访问是由KMS和密钥存储上的单独权限集控制的，因此这不会造成安全威胁。

### 3.3 Key Management Server, KeyProvider, EDEKs

KMS是代表HDFS守护进程、客户端与后备密钥存储对接的代理。后辈秘钥存储与KMS都实现了Hadoop KeyProvider API。更多信息参见 [KMS documentation](https://hadoop.apache.org/docs/current/hadoop-kms/index.html) 文档。

在KeyProvider中，每个加密秘钥都有一个唯一的名称，因为秘钥可以滚动，所以一个密钥可以具有多个密钥版本，其中每个密钥版本都有自己的密钥组成(加密和解密过程中使用的实际秘密字节)。可以通过秘钥名称，返回秘钥最新版本或特定秘钥版本来获取加密秘钥。

KMS实现了附加功能，允许创建和解密加密的加密密钥(EEK，双重加密)，EEK的创建和解密完全在KMS上进行。 重要的是，请求创建或解密EEK的客户端永远不会处理EEK的加密密钥。 要创建新的EEK，KMS会生成一个新的随机密钥，并使用指定的密钥对其进行加密，然后将EEK返回给客户端。 为了解密EEK，KMS会检查用户是否有权使用加密密钥，并使用它来解密EEK，然后返回解密的加密密钥。

在HDFS加密的上下文中，EEK是加密的数据加密密钥(EDEK)，其中数据加密密钥(DEK)用于加密和解密文件数据。通常，密钥库配置为仅允许最终用户访问用于加密DEK的密钥。 这意味着EDEK可以由HDFS安全地存储和处理，因为HDFS用户将无法访问未加密的加密密钥。

## 4. 配置

一个必要的先决条件是KMS的实例，以及KMS的后备密钥存储。 有关更多信息，请参见[KMS documentation](https://hadoop.apache.org/docs/current/hadoop-kms/index.html)。

一旦KMS启动，而且nameNode与HDFS客户端被正确配置，管理员可以使用hadoop密钥和hdfs加密命令行工具来创建加密密钥并设置新的加密区域。 可以使用distcp等工具将现有数据复制到新的加密区域中，从而对现有数据进行加密。

### 4.1 配置集群KeyProvider

**hadoop.security.key.provider.path**

KeyProvider用于在读取和写入加密区域时，使用加密密钥进行交互。HDFS客户端将使用通过getServerDefaults从Namenode返回的提供程序路径。 如果namenode不支持返回密钥提供者uri，则将使用客户端的conf。

### 4.2 选择加密算法和解码器

**hadoop.security.crypto.codec.classes.EXAMPLECIPHERSUITE**

给定密码编解码器的前缀，包含逗号分隔的给定密码编解码器的实现类列表（例如，EXAMPLECIPHERSUITE）。 如果可用，将使用第一个实现，其他则是后备。

**hadoop.security.crypto.codec.classes.aes.ctr.nopadding**

默认：org.apache.hadoop.crypto.OpensslAesCtrCryptoCodec, org.apache.hadoop.crypto.JceAesCtrCryptoCodec

逗号分隔的AES/CTR/NoPadding加密编解码器实现列表。 如果可用，将使用第一个实现，其他则是后备。

**hadoop.security.crypto.cipher.suite**

默认：AES/CTR/NoPadding

编解码器的密码套件。

**hadoop.security.crypto.jce.provider**

默认：None

CryptoCodec使用的JCE provider程序名。

**hadoop.security.crypto.buffer.size**

默认：8192

CryptoInputStream和CryptoOutputStream使用的缓冲区大小。

### 4.3 NameNode配置

**dfs.namenode.list.encryption.zones.num.responses**

默认：100

列出加密区域时，批量返回最大区域数。 批量增量获取列表可提高nameNode的性能。

## 5. `crypto`命令行

### 5.1 创建区域

用法：`-createZone -keyName <keyName> -path <path>`

创建新的加密区域

| 命令行参数 | 描述                                                         |
| ---------- | ------------------------------------------------------------ |
| path       | 创建加密区域的路径，它必须是一个空目录。 在此路径下设置了垃圾目录 |
| keyname    | 用于加密区域的密钥名称。 不支持大写键名                      |

### 5.2 列举加密区域

用法：`-listZones`

列出所有加密区域。 需要超级用户权限。

### 5.3 provisionTrash

用法：`-provisionTrash -path <path>`

为加密区域设置回收站目录

### 5.4 getFileEncryptionInfo

用法：`-getFileEncryptionInfo -path <path>`

获取文件加密信息。可以用于确定文件是否已经加密，以及密钥名称/密钥版本。

### 5.5 reencryptZone

用法：`-reencryptZone <action> -path <zone>`

通过遍历加密区域并调用KeyProvider的`reencryptEncryptedKeys`接口，使用密钥提供者中的最新版本的加密区域密钥对所有文件的EDEK进行批量重新加密。重新加密需要超级用户权限。

注意，由于快照具有不变性，因此重新加密不适用于快照。

| 参数   | 描述                                    |
| ------ | --------------------------------------- |
| action | 重新加密动作执行，必须为-start或-cancel |
| path   | 加密区域的根目录                        |

重新加密是HDFS中仅在NameNode上的操作，因此可能会对NameNode施加大量负载。 可以更改以下配置以控制对NameNode的压力，具体取决于对集群的吞吐量影响。

| 配置                                                | 描述                                                         |
| --------------------------------------------------- | ------------------------------------------------------------ |
| dfs.namenode.reencrypt.batch.size                   | 每批发送到KMS重加密的EDEK数量。在获取nameSpace读写锁时，将处理每个批次，并且在批次之间会发生限制。 |
| dfs.namenode.reencrypt.throttle.limit.handler.ratio | 重新加密期间要保留的读取锁的比率。 1.0表示没有限制，0.5表示最多可以将readlock保留其总处理时间的50％，负值或0无效。 |
| dfs.namenode.reencrypt.throttle.limit.updater.ratio | 重新加密期间要保留的写锁的比率。 1.0表示没有限制，0.5表示最多可以将readlock保留其总处理时间的50％，负值或0无效。 |

### 5.6 listReencryptionStatus

用法：`-listReencryptionStatus`

列出所有加密区域的重新加密信息，需要超级用户权限。

## 6. 使用实例

这些说明假定您以适当的普通用户或HDFS超级用户身份运行。

```sh
# As the normal user, create a new encryption key
hadoop key create mykey

# As the super user, create a new empty directory and make it an encryption zone
hadoop fs -mkdir /zone
hdfs crypto -createZone -keyName mykey -path /zone

# chown it to the normal user
hadoop fs -chown myuser:myuser /zone

# As the normal user, put a file in, read it out
hadoop fs -put helloWorld /zone
hadoop fs -cat /zone/helloWorld

# As the normal user, get encryption information from the file
hdfs crypto -getFileEncryptionInfo -path /zone/helloWorld

# console output: {cipherSuite: {name: AES/CTR/NoPadding, algorithmBlockSize: 16}, cryptoProtocolVersion: CryptoProtocolVersion{description='Encryption zones', version=1, unknownValue=null}, edek: 2010d301afbd43b58f10737ce4e93b39, iv: ade2293db2bab1a2e337f91361304cb3, keyName: mykey, ezKeyVersionName: mykey@0}
```

## 7. Distcp注意事项

### 7.1 以超级管理员身份运行

distcp的一种常见用例是在群集之间复制数据，以进行备份和灾难恢复。这通常由作为HDFS超级用户的群集管理员执行。

为了在使用HDFS加密时启用相同的工作流程，引入了新的虚拟路径前缀`/.reserved/raw/`，该超级路径使超级用户可以直接访问文件系统中的基础块数据。这使超级用户无需访问加密密钥即可分发数据，并且还避免了解密和重新加密数据的开销。这也意味着源数据和目标数据将是逐字节相同的，如果使用新的EDEK对数据进行了重新加密，那将不是正确的。

使用`/.reserved/raw`分发加密数据时，请务必使用-px标志保留扩展属性。这是因为加密的文件属性(例如EDEK)是通过`/.reserved/raw`中的扩展属性公开的，必须保留这些属性才能解密文件。这意味着，如果distcp在加密区域根目录或更高级别启动，则它将在目标位置自动创建一个加密区域（如果尚不存在）。但是，仍然建议管理员首先在目标群集上创建相同的加密区域，以免发生任何潜在的事故。

### 7.2  复制到加密区域

默认情况下，distcp比较文件系统提供的校验和，以验证数据已成功复制到目标。 从未加密或加密位置复制到另一加密位置时，文件系统校验和将不匹配，因为底层基础块数据不同。这是由于使用新的EDEK在目标位置对文件重新加密。在这种情况下，使用`-skipcrccheck`和`-update distcp`，以避免验证校验和。

## 8. Rename and Trash considerations

HDFS限制跨加密区域边界的文件和目录重命名。这包括将加密的文件/目录重命名为未加密的目录(例如，hdfs dfs mv /zone/encryptedFile /home/bob)，将未加密的文件或目录重命名为加密区域(例如，hdfs dfs mv/home/bob /unEncryptedFile/zone)。并在两个不同的加密区域(例如 hdfs dfs mv /home/alice/zone1/foo    /home/alice/zone2)之间重命名。在这些示例中/zone，/home/alice/zone1和/home/alice/zone2是加密区域，而/home/bob不是。仅当源路径和目标路径在同一加密区域中，或者两个路径都未加密(不在任何加密区域中)时，才允许重命名。

此限制增强了安全性，并大大简化了系统管理。加密区域下的所有文件EDEK均使用加密区域密钥加密。因此，如果加密区域密钥受到破坏，则重要的是识别所有易受攻击的文件并重新对其进行加密。如果可以将最初在加密区域中创建的文件重命名为文件系统中的任意位置，则从根本上讲这是困难的。

为了遵守上述规则，每个加密区域在区域目录下都有自己的`.Trash`目录。例如，在`hdfs dfs rm /zone/encryptedFile`之后，encryptedFile将移至`/zone/.Trash`，而不是用户主目录下的`.Trash`目录。删除整个加密区域后，区域目将移至用户主目录下的.Trash目录。

如果加密区域是根目录(例如/directory)，则根目录的回收站路径为/.Trash，而不是用户主目录下的.Trash目录，并且重命名子目录或子文件的行为，根目录将与常规加密区域(例如本节顶部提到的/zone)中的行为保持一致。

Hadoop 2.8.0之前的crypto命令不会自动设置.Trash目录。如果在Hadoop 2.8.0之前创建了加密区域，然后将集群升级到Hadoop 2.8.0或更高版本，则可以使用-provisionTrash选项(例如hdfs crypto -provisionTrash -path /zone)设置回收站目录。

## 9. 攻击方式

### 9.1 硬件访问漏洞

假设攻击者已从群集机器，即dataNode和nameNode，获得了对硬盘驱动器的物理访问权限。

1. 访问包含数据加密密钥的进程的交换文件
	- 就其本身而言，这不会显示明文，因为它还需要访问加密的块文件
	- 可以通过禁用交换，使用加密交换或使用mlock防止密钥被交换来缓解这种情况
2. 访问加密的块文件
	- 就其本身而言，这不会公开明文，因为它还需要访问DEK

### 9.2 Root访问漏洞

假设攻击者已获得对群集机器，即dataNode和nameNode的root shell访问权限。由于恶意的root用户可以访问持有加密密钥和明文的进程的内存状态，因此无法利用HDFS解决其中的许多漏洞。 对于这些攻击，唯一的缓解技术是仔细限制和监视根外壳程序访问。

1. 访问加密块文件
	- 就其本身而言，这不会公开明文，因为它还需要访问加密密钥
2. 转储客户端进程的内存以获得DEK，委托令牌，明文
	- 无法缓解
3. 记录网络流量监控密密钥和传输中的加密数据
	- 就其本身而言，没有EDEK加密密钥不足以读取明文
4. 转储datanode进程的内存以获取加密的块数据
	- 就其本身而言，没有DEK不足以读取明文
5. 转储namenode进程的内存以获取加密的数据加密密钥
	- 就其本身而言，如果没有EDEK的加密密钥和加密的块文件，则不足以读取明文

### 9.3 HDFS管理员漏洞

假设攻击者已经破坏了HDFS，但没有root或hdfs用户外壳程序访问权限。

1. 访问加密的块文件
	- 就其本身而言，如果没有EDEK和EDEK加密密钥，则不足以读取明文
2. 通过-fetchImage访问加密区域和加密文件元数据(包括加密数据加密密钥)
	- 就其本身而言，没有EDEK加密密钥不足以读取明文。

### 9.4 恶意用户漏洞

恶意用户可以收集他们有权访问的文件的密钥，并在以后使用它们解密那些文件的加密数据。 由于用户有访问这些文件的权限，因此他们已经可以访问文件内容。 这可以通过定期的密钥滚动策略来缓解。 密钥滚动后通常需要reencryptZone命令，以确保现有文件上的EDEK使用新版本的密钥。

下面列出了完成密钥滚动与重新加密的手动步骤。 需要以适当的管理员身份或HDFS超级用户身份运行：

```sh
# As the key admin, roll the key to a new version
hadoop key roll exposedKey

# As the super user, re-encrypt the encryption zone. Possibly list zones first.
hdfs crypto -listZones
hdfs crypto -reencryptZone -start -path /zone

# As the super user, periodically check the status of re-encryption
hdfs crypto -listReencryptionStatus

# As the super user, get encryption information from the file and double check it's encryption key version
hdfs crypto -getFileEncryptionInfo -path /zone/helloWorld

# console output: {cipherSuite: {name: AES/CTR/NoPadding, algorithmBlockSize: 16}, cryptoProtocolVersion: CryptoProtocolVersion{description='Encryption zones', version=2, unknownValue=null}, edek: 2010d301afbd43b58f10737ce4e93b39, iv: ade2293db2bab1a2e337f91361304cb3, keyName: exposedKey, ezKeyVersionName: exposedKey@1}
```



