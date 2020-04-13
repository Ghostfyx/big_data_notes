# Offline Image Viewer Guide

 ## 1. 概述

Offline Image Viewer(离线镜像文件查看器，oiv)是一种可以将hdfs fsimage文件的内容转换为人类可读的格式，并提供只读的WebHDFS API，以允许脱机分析和检查Hadoop群集的命名空间。该工具能够相对快速地处理非常大的Fsimage文件。该工具处理Hadoop 2.4及更高版本的设计格式。 如果要处理较旧版本的设计格式，则可以使用Hadoop 2.3的Offline Image Viewer或oiv_legacy Command。

离线镜像文件查看器提供了以下几个输出器：

- Web：默认的输出处理器。 它启动一个HTTP服务器，该服务器提供只读的WebHDFS API。 用户可以使用HTTP REST API以交互方式查看命名空间。 它不支持安全模式，也不支持HTTPS。
- XML：XML创建fsimage的XML文档，并包括fsimage中的所有信息。 该处理器的输出适合使用XML工具进行自动处理和分析。 由于XML语法的冗长性，该处理器还将生成最大量的输出。
- FileDistribution：用于分析命名空间映像中文件大小的工具。为了运行该工具，应通过指定maxSize和一个步骤来定义整数范围[0，maxSize]。整数范围分为大小为段的段：$[0,s[1],\dots,s[n-1],maxSize]$，处理器计算系统中每个段中有多少文件$[s[i-1],s[i])$。请注意，大于maxSize的文件始终属于最后一段。 默认情况下，输出文件的格式设置为制表符，分隔两列表：Size和NumFiles。 其中Size代表该段的开始，numFiles是该映像中属于该段的图像文件数量。过指定选项-format，将以易于理解的方式格式化输出文件，而不是“大小”列中显示的字节数。 此外，“大小”列将更改为“大小范围”列。
- Delimited：生成一个文本文件，其中包含由分隔符分隔的inode和创建中inode的所有元素。 默认定界符为\ t，可以通过-delimiter参数更改界定符。
- ReverseXML ：这与XML处理器相反。 它从XML文件重建fsimage。 使用此处理器，可以轻松创建fsimage进行测试，并在损坏时手动编辑fsimage。

## 2. 使用方式

### 2.1 web处理器

Web处理器启动HTTP服务器，该服务器提供只读WebHDFS API。 用户可以通过-addr选项指定要监听的地址(默认为localhost：5978)。

```
hdfs oiv -i fsimage
14/04/07 13:25:14 INFO offlineImageViewer.WebImageViewer: WebImageViewer
started. Listening on /127.0.0.1:5978. Press Ctrl+C to stop the viewer.
```

用户可以通过以下shell命令访问查看器并获取fsimage的信息：

```
hdfs dfs -ls webhdfs://127.0.0.1:5978/

Found 2 items
drwxrwx--* -root supergroup 0 2014-03-26 20:16 webhdfs://127.0.0.1:5978/tmp
drwxr-xr-x -root supergroup 0 2014-03-31 14:08 webhdfs://127.0.0.1:5978/user
```

要获取所有文件和目录的信息，可以使用以下命令：

```
hdfs dfs -ls -R webhdfs://127.0.0.1:5978/
```

用户还可以通过HTTP REST API获得JSON格式的FileStatuses：

```
curl -i http://127.0.0.1:5978/webhdfs/v1/?op=liststatus

HTTP/1.1 200 OK
Content-Type: application/json
Content-Length: 252

{"FileStatuses":{"FileStatus":[
{"fileId":16386,"accessTime":0,"replication":0,"owner":"theuser","length":0,"permission":"755","blockSize":0,"modificationTime":1392772497282,"type":"DIRECTORY","group":"supergroup","childrenNum":1,"pathSuffix":"user"}
]}}
```

web处理器现在支持以下命令选项：

- [LISTSTATUS](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#List_a_Directory)   返回json格式的所有文件状态
- [GETFILESTATUS](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Status_of_a_FileDirectory)   
- [GETACLSTATUS](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Get_ACL_Status)
- [GETXATTRS](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Get_an_XAttr)
- [LISTXATTRS](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#List_all_XAttrs)
- [CONTENTSUMMARY](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-hdfs/WebHDFS.html#Get_Content_Summary_of_a_Directory)