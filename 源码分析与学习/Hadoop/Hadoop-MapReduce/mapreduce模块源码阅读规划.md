## client-core模块源码分析时间序列

重点对其中的类进行分析。从Driver驱动器程序配置Job，到提交Job，读取输入数据，执行数据处理，输出数据整个流程串行对源码分析。

| 类名                                                        | 所属模块                          | 剖析日期   |
| ----------------------------------------------------------- | --------------------------------- | ---------- |
| org.apache.hadoop.mapreduce.Job                             | hadoop-mapreduce-client-core      | 2020/04/26 |
| org.apache.hadoop.mapreduce.JobSubmitter                    | hadoop-mapreduce-client-core      | 2020/04/27 |
| org.apache.hadoop.mapreduce.Cluster                         | hadoop-mapreduce-client-core      | 2020/04/28 |
| org.apache.hadoop.mapreduce.protocol.ClientProtocolProvider | hadoop-mapreduce-client-core      | 2020/04/28 |
| org.apache.hadoop.mapreduce.protocol.ClientProtocol         | hadoop-mapreduce-client-core      | 2020/04/29 |
| org.apache.hadoop.mapred.YARNRunner                         | hadoop-mapreduce-client-jobclient | 2020/04/29 |
| org.apache.hadoop.mapred.LocalJobRunner                     | hadoop-mapreduce-client-common    | 2020/04/29 |

| 日期       | 学习源码                       | 描述                   |
| ---------- | ------------------------------ | ---------------------- |
| 2020-06-04 | Mapper.class, MapContext.class | MapReduce Mapper类学习 |