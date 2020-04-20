# MapReduce Commands Guide

 ## 1. 概述

所有的mapreduce命令都由`bin/mapred`脚本调用。 运行不带任何参数的mapred脚本将打印所有命令的描述。

用法：`mapred [SHELL_OPTIONS] COMMAND [GENERIC_OPTIONS] [COMMAND_OPTIONS]`

Hadoop有一个选项解析框架，该框架使用解析通用选项以及运行类。

| 选项                    | 描述                                                         |
| ----------------------- | ------------------------------------------------------------ |
| SHELL_OPTIONS           | 通用shell选项，详情参考[Hadoop Commands Reference](https://hadoop.apache.org/docs/stable/hadoop-project-dist/hadoop-common/CommandsManual.html#Shell_Options) |
| GENERIC_OPTIONS         | 多个命令组成的一组通用选项                                   |
| COMMAND COMMAND_OPTIONS | 命令已分组为用户命令和管理命令                               |

## 2. User Command

对hadoop集群用户的命令。

### 2.1 archive

创建一个hadoop存档。 有关更多信息，请参见[Hadoop Archives Guide](https://hadoop.apache.org/docs/stable/hadoop-archives/HadoopArchives.html)。

### 2.2 archive-logs

将YARN聚合日志移动到Hadoop归档中以减少HDFS中文件数量的工具。 请详参[Hadoop Archive Logs Guide](https://hadoop.apache.org/docs/stable/hadoop-archive-logs/HadoopArchiveLogs.html)。

### 2.3 classpath

用法：`yarn classpath [--glob |--jar <path> |-h |--help]`

| 选项           | 描述         |
| -------------- | ------------ |
| --glob         | 扩展通配符   |
| `--jar` *path* |              |
| -h,--help      | 打印帮助命令 |

### 2.4 distcp

递归复制文件或目录。 有关更多信息，请参见[Hadoop DistCp Guide](https://hadoop.apache.org/docs/stable/hadoop-distcp/DistCp.html)。

### 2.5 job

与MapReduce Jobs交互的命令。

用法：`mapred job |[GENERIC_OPTIONS]|[-submit]|[-status]|[-counter]|[-kill]|[-events <#-of-events>]|[-history [all] [-outfile] [-format]]|[-list [all]]|[-kill-task]|[-fail-task]|[-set-priority]|[-list-active-trackers]|[-list-blacklisted-trackers]|[-list-attempt-ids][-logs][-config]`

| 选型                                                         | 描述                                                         |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| -submit *job-file*                                           | 提交作业                                                     |
| -status *job-id*                                             | 打印map和reduce完成比例和所有作业计数器                      |
| -counter *job-id* *group-name* *counter-name*                | 打印计数器值                                                 |
| -kill *job-id*                                               | 杀死作业                                                     |
| -events *job-id* *from-event-#* *#-of-events*                | 打印给定范围内jobtracker接收到的事件的详细信息               |
| -history [all] *jobHistoryFilejobId* [-outfile *file*] [-format *humanjson*] | 打印作业详细信息，失败和终止的任务详细信息。通过指定[all]选项，可以查看有关作业的更多详细信息，例如成功的任务，为每个任务进行的任务尝试，任务计数器等。 可以指定一个可选的文件输出路径(而不是stdout)。 格式默认为人类可读，但也可以使用[-format]选项更改为JSON。 |
| -list [all]                                                  | 打印所有已经完成的作业，-list all 打印所有作业               |
| -kill-task *task-id*                                         | 杀死任务。 杀死的任务不计入失败计数                          |
| -fail-task *task-id*                                         | 失败任务， 失败的任务计入失败计数                            |
| -set-priority *job-id* *priority*                            | 更改作业的优先级。 允许的优先级值为VERY_HIGH，HIGH，NORMAL，LOW，VERY_LOW |
| -list-active-trackers                                        | 显示集群中所有活跃NodeManager                                |
| -list-blacklisted-trackers                                   | 列出集群中黑名单中的任务跟踪器。 基于MRv2的群集不支持此命令。 |
| -list-attempt-ids *job-id* *task-type* *task-state*          | 根据任务类型和给定状态列出推测重试ID。 任务类型为REDUCE、MAP。 任务状态的为正在运行，挂起，完成，失败，已终止。 |
| -logs *job-id* *task-attempt-id*                             |                                                              |
| -config *job-id* *file*                                      | 下载指定任务的配置文件                                       |

### 2.6 pipes

运行管道作业。

用法：`mapred pipes [-conf <path>] [-jobconf <key=value>, <key=value>, ...] [-input <path>] [-output <path>] [-jar <jar file>] [-inputformat <class>] [-map <class>] [-partitioner <class>] [-reduce <class>] [-writer <class>] [-program <executable>] [-reduces <num>]`

| COMMAND_OPTION                       | Description                        |
| :----------------------------------- | :--------------------------------- |
| -conf *path*                         | Configuration for job              |
| -jobconf *key=value*, *key=value*, … | Add/override configuration for job |
| -input *path*                        | Input directory                    |
| -output *path*                       | Output directory                   |
| -jar *jar file*                      | Jar filename                       |
| -inputformat *class*                 | InputFormat class                  |
| -map *class*                         | Java Map class                     |
| -partitioner *class*                 | Java Partitioner                   |
| -reduce *class*                      | Java Reduce class                  |
| -writer *class*                      | Java RecordWriter                  |
| -program *executable*                | Executable URI                     |
| -reduces *num*                       | Number of reduces                  |

### 2.7 queue

用于交互和查看作业队列信息。

用法：`mapred queue [-list] | [-info <job-queue-name> [-showJobs]] | [-showacls]`

| COMMAND_OPTION                     | Description                                                  |
| :--------------------------------- | :----------------------------------------------------------- |
| -list                              | 获取系统中配置的作业队列列表。 以及与作业队列关联的调度信息  |
| -info *job-queue-name* [-showJobs] | 显示作业队列信息和特定作业队列的关联调度信息。 如果存在`-showJobs`选项，则会显示提交到特定作业队列的作业列表 |
| -showacls                          | 显示当前用户允许的队列名称和关联的队列操作。 该列表仅包含用户有权访问的那些队列 |

### 2.8 version

打印版本号。

用法：`mapred version`

## 3. 管理员命令

Hadoop集群管理员命令。

### 3.1 历史服务器

开启JobHistoryServer。

用法：`mapred historyserver`

### 3.2 hsadmin

运行MapReduce hsadmin客户端以执行JobHistoryServer管理命令。

用法：`mapred hsadmin [-refreshUserToGroupsMappings] | [-refreshSuperUserGroupsConfiguration] | [-refreshAdminAcls] | [-refreshLoadedJobCache] | [-refreshLogRetentionSettings] | [-refreshJobRetentionSettings] | [-getGroups [username]] | [-help [cmd]]`

| COMMAND_OPTION                       | Description                                                  |
| :----------------------------------- | :----------------------------------------------------------- |
| -refreshUserToGroupsMappings         | Refresh user-to-groups mappings                              |
| -refreshSuperUserGroupsConfiguration | Refresh superuser proxy groups mappings                      |
| -refreshAdminAcls                    | Refresh acls for administration of Job history server        |
| -refreshLoadedJobCache               | Refresh loaded job cache of Job history server               |
| -refreshJobRetentionSettings         | Refresh job history period, job cleaner settings             |
| -refreshLogRetentionSettings         | Refresh log retention period and log retention check interval |
| -getGroups [username]                | Get the groups which given user belongs to                   |
| -help [cmd]                          | Displays help for the given command or all commands if none is specified |

### 3.3 frameworkuploader

收集框架jar并将其作为tarball上传到HDFS。

用法：`mapred frameworkuploader -target <target> [-fs <filesystem>] [-input <classpath>] [-blacklist <list>] [-whitelist <list>] [-initialReplication <num>] [-acceptableReplication <num>] [-finalReplication <num>] [-timeout <seconds>] [-nosymlink]`

| COMMAND_OPTION               | Description                                                  |
| :--------------------------- | :----------------------------------------------------------- |
| -input *classpath*           | This is the input classpath that is searched for jar files to be included in the tarball. |
| -fs *filesystem*             | The target file system. Defaults to the default filesystem set by fs.defaultFS. |
| -target *target*             | This is the target location of the framework tarball, optionally followed by a # with the localized alias. An example would be /usr/lib/framework.tar#framework. Make sure the target directory is readable by all users but it is not writable by others than administrators to protect cluster security. |
| -blacklist *list*            | This is a comma separated regex array to filter the jar file names to exclude from the class path. It can be used for example to exclude test jars or Hadoop services that are not necessary to localize. |
| -whitelist *list*            | This is a comma separated regex array to include certain jar files. This can be used to provide additional security, so that no external source can include malicious code in the classpath when the tool runs. |
| -nosymlink                   | This flag can be used to exclude symlinks that point to the same directory. This is not widely used. For example, `/a/foo.jar` and a symlink `/a/bar.jar` that points to `/a/foo.jar` would normally add `foo.jar` and `bar.jar` to the tarball as separate files despite them actually being the same file. This flag would make the tool exclude `/a/bar.jar` so only one copy of the file is added. |
| -initialReplication *num*    | This is the replication count that the framework tarball is created with. It is safe to leave this value at the default 3. This is the tested scenario. |
| -finalReplication *num*      | The uploader tool sets the replication once all blocks are collected and uploaded. If quick initial startup is required, then it is advised to set this to the commissioned node count divided by two but not more than 512. |
| -acceptableReplication *num* | The tool will wait until the tarball has been replicated this number of times before exiting. This should be a replication count less than or equal to the value in `finalReplication`. This is typically a 90% of the value in `finalReplication` to accomodate failing nodes. |
| -timeout *seconds*           | A timeout in seconds to wait to reach `acceptableReplication` before the tool exits. The tool logs an error otherwise and returns. |