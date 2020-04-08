# Offline Edits Viewer Guide

 ## 1. 概述

离线编辑日志浏览器(Offline Edits Viewer)是用于解析编辑日志文件(Edits Log)的工具。处理器对于在不同格式之间进行转换非常有用，包括将原始二进制格式转换为更易于阅读和编辑的XML格式。

该工具可以解析(Hadoop 0.19)及更高版本的编辑日志格式。 该工具仅在文件上运行，不需要运行Hadoop集群。

支持以下输入格式：

- 二进制(binary)：Hadoop内部使用的本机二进制格式
- xml：由XML处理器产生的XML格式，适用于文件名具有.xml扩展名的文件

注意：不允许XML/二进制格式输入文件由同一类型的处理器处理。

离线编辑日志浏览器供了以下几个输出处理器(除非另有说明，否则可以将处理器的输出转换回原始编辑文件):

- 二进制：Hadoop内部使用的本机二进制格式
- xml：XML格式
- stats：打印出统计信息，无法将其转换回Edits文件

## 2. 用法

### 2.1 XML处理器

XML处理器可以创建一个包含编辑日志信息的XML文件。用户可以通过`-i`和`-o`命令行指定输入和输出文件。

```
hdfs oev -p xml -i edits -o edits.xml
```

XML处理器是离线编辑日志浏览器的默认处理器，可以使用以下命令：

```
hdfs oev -i edits -o edits.xml
```

编辑日志XML输出结果如下：

```xml
<?xml version="1.0" encoding="UTF-8"?>
   <EDITS>
     <EDITS_VERSION>-64</EDITS_VERSION>
     <RECORD>
       <OPCODE>OP_START_LOG_SEGMENT</OPCODE>
       <DATA>
         <TXID>1</TXID>
       </DATA>
     </RECORD>
     <RECORD>
       <OPCODE>OP_UPDATE_MASTER_KEY</OPCODE>
       <DATA>
         <TXID>2</TXID>
         <DELEGATION_KEY>
           <KEY_ID>1</KEY_ID>
           <EXPIRY_DATE>1487921580728</EXPIRY_DATE>
           <KEY>2e127ca41c7de215</KEY>
         </DELEGATION_KEY>
       </DATA>
     </RECORD>
     <RECORD>
   ...remaining output omitted...
```

### 2.2 二进制处理器

二进制处理器与XML处理器相反。 用户可以通过`-i`和`-o`命令行指定输入XML文件和输出文件。

```
hdfs oev -p binary -i edits.xml -o edits
```

这将从XML文件重新构建编辑日志文件。

### 2.3 Stats处理器

统计处理器用于汇总编辑日志文件中包含的操作码计数。 用户可以通过-p选项指定此处理器。

```
hdfs oev -p stats -i edits -o edits.stats
```

该处理器的输出结果应类似于以下输出：

```
VERSION                             : -64
OP_ADD                         (  0): 8
OP_RENAME_OLD                  (  1): 1
OP_DELETE                      (  2): 1
OP_MKDIR                       (  3): 1
OP_SET_REPLICATION             (  4): 1
OP_DATANODE_ADD                (  5): 0
OP_DATANODE_REMOVE             (  6): 0
OP_SET_PERMISSIONS             (  7): 1
OP_SET_OWNER                   (  8): 1
OP_CLOSE                       (  9): 9
OP_SET_GENSTAMP_V1             ( 10): 0
...some output omitted...
OP_APPEND                      ( 47): 1
OP_SET_QUOTA_BY_STORAGETYPE    ( 48): 1
OP_ADD_ERASURE_CODING_POLICY   ( 49): 0
OP_ENABLE_ERASURE_CODING_POLICY  ( 50): 1
OP_DISABLE_ERASURE_CODING_POLICY ( 51): 0
OP_REMOVE_ERASURE_CODING_POLICY  ( 52): 0
OP_INVALID                     ( -1): 0
```

输出格式设置为以冒号分隔的包含两列的表：OpCode与OpCodeCount。每个操作码对应于NameNode中的特定操作。

## 3. 命令选项

| 选项                                  | 描述                                                         |
| ------------------------------------- | ------------------------------------------------------------ |
| [`-i` ; `--inputFile`] *input file*   | 指定要处理的输入编辑日志文件。 Xml(不区分大小写)扩展名表示XML格式，否则假定为二进制格式。 |
| [`-o` ; `--outputFile`] *output file* | 如果指定的输出处理器生成一个文件名，则指定输出文件名。 如果指定的文件已经存在，覆盖原有文件 |
| [`-p` ; `--processor`] *processor*    | 指定要应用于编辑日志文件的处理器。 当前有效的选项是binary，xml(默认)和stats |
| [`-v` ; `--verbose`]                  | 将处理器的输入和输出文件名以及管道输出打印到控制台以及指定文件。 在非常大的文件上，这可能会使处理时间增加一个数量级 |
| [`-f` ; `--fix-txids`]                | 在输入中重新编号事务ID，以确保没有空格或无效的事务ID         |
| [`-r` ; `--recover`]                  | 读取二进制编辑日志时，请使用恢复模式。 有机会跳过编辑日志的损坏部分 |
| [`-h` ; `--help`]                     | 显示工具用法和帮助信息，然后退出                             |

## 4. 案例研究：Hadoop集群恢复

假设hadoop群集出现问题并且编辑文件已损坏，通过Offline Edits Viewer操作可以保存至少一部分正确的编辑文件。通过将二进制编辑转换为XML，手动进行编辑，然后再将其转换回二进制来完成。 最常见的问题是编辑日志文件缺少结束记录(具有opCode -1的记录)。该工具应识别出该错误，并且应正确关闭XML文件。

如果XML文件中没有结束记录，则可以在最后一个正确的记录之后添加一个。opCode -1的记录之后的所有内容都将被忽略。

关闭记录的示例(使用opCode -1)：

```xml
<RECORD>
  <OPCODE>-1</OPCODE>
  <DATA>
  </DATA>
</RECORD>
```

