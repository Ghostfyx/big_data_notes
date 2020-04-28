# MapReduce-client-core模块详解

## 1. 目录结构

```
src
|__main
    |__avro org.apache.hadoop.mapreduce.jobhistory avro模板文件
    |__java 源代码
       |___org.apache.hadoop
       		|___filecache
       		|___mapred 旧版mapreduce API(不必看)
       		|___mapreduce 新版mapreduce API
    |__resource 配置资源，包括mapred-default.xml
|___site 相关文档
|___test
```

## 2. avro目录

仅有一个文件Events.avpr，使用Apache Avro序列化框架定义事件的序列化/反序列化。

```json
{"namespace": "org.apache.hadoop.mapreduce.jobhistory",
 "protocol": "Events",

 "types": [

     {"type": "record", "name": "JhCounter",
      "fields": [
          {"name": "name", "type": "string"},
          {"name": "displayName", "type": "string"},
          {"name": "value", "type": "long"}
      ]
     },

     {"type": "record", "name": "JhCounterGroup",
      "fields": [
          {"name": "name", "type": "string"},
          {"name": "displayName", "type": "string"},
          {"name": "counts", "type": {"type": "array", "items": "JhCounter"}}
      ]
     },

     {"type": "record", "name": "JhCounters",
      "fields": [
          {"name": "name", "type": "string"},
          {"name": "groups", "type": {"type": "array", "items": "JhCounterGroup"}}
      ]
     },

     {"type": "record", "name": "JobFinished",
      "fields": [
          {"name": "jobid", "type": "string"},
          {"name": "finishTime", "type": "long"},
          {"name": "finishedMaps", "type": "int"},
          {"name": "finishedReduces", "type": "int"},
          {"name": "failedMaps", "type": "int"},
          {"name": "failedReduces", "type": "int"},
          {"name": "totalCounters", "type": "JhCounters"},
          {"name": "mapCounters", "type": "JhCounters"},
          {"name": "reduceCounters", "type": "JhCounters"},
          {"name": "killedMaps", "type": "int", "default": -1},
          {"name": "killedReduces", "type": "int", "default": -1}
      ]
     },

     {"type": "record", "name": "JobInited",
      "fields": [
          {"name": "jobid", "type": "string"},
          {"name": "launchTime", "type": "long"},
          {"name": "totalMaps", "type": "int"},
          {"name": "totalReduces", "type": "int"},
          {"name": "jobStatus", "type": "string"},
          {"name": "uberized", "type": "boolean"}
      ]
     },

     {"type": "record", "name": "AMStarted",
      "fields": [
          {"name": "applicationAttemptId", "type": "string"},
          {"name": "startTime", "type": "long"},
          {"name": "containerId", "type": "string"},
          {"name": "nodeManagerHost", "type": "string"},
          {"name": "nodeManagerPort", "type": "int"},
          {"name": "nodeManagerHttpPort", "type": "int"}
      ]
     },

     {"type": "record", "name": "JobSubmitted",
      "fields": [
          {"name": "jobid", "type": "string"},
          {"name": "jobName", "type": "string"},
          {"name": "userName", "type": "string"},
          {"name": "submitTime", "type": "long"},
          {"name": "jobConfPath", "type": "string"},
          {"name": "acls", "type": {"type": "map",
                                    "values": "string"
                                   }
          },
          {"name": "jobQueueName", "type": "string"},
          {"name": "workflowId", "type": ["null","string"], "default": null},
          {"name": "workflowName", "type": ["null","string"], "default": null},
          {"name": "workflowNodeName", "type": ["null","string"], "default": null},
          {"name": "workflowAdjacencies", "type": ["null","string"], "default": null},
          {"name": "workflowTags", "type": ["null","string"], "default": null}
      ]
     },

     {"type": "record", "name": "JobInfoChange",
      "fields": [
          {"name": "jobid", "type": "string"},
          {"name": "submitTime", "type": "long"},
          {"name": "launchTime", "type": "long"}
      ]
     },

     {"type": "record", "name": "JobPriorityChange",
      "fields": [
          {"name": "jobid", "type": "string"},
          {"name": "priority", "type": "string"}
      ]
     },

     {"type": "record", "name": "JobStatusChanged",
      "fields": [
          {"name": "jobid", "type": "string"},
          {"name": "jobStatus", "type": "string"}
      ]
     },

     {"type": "record", "name": "JobQueueChange",
      "fields": [
          {"name": "jobid", "type": "string"},
          {"name": "jobQueueName", "type": "string"}
      ]
     },

     {"type": "record", "name": "JobUnsuccessfulCompletion",
      "fields": [
          {"name": "jobid", "type": "string"},
          {"name": "finishTime", "type": "long"},
          {"name": "finishedMaps", "type": "int"},
          {"name": "finishedReduces", "type": "int"},
          {"name": "jobStatus", "type": "string"},
          {"name": "diagnostics", "type": ["null","string"], "default": null},
          {"name": "failedMaps", "type": "int", "default": -1},
          {"name": "failedReduces", "type": "int",  "default": -1},
          {"name": "killedMaps", "type": "int",  "default": -1},
          {"name": "killedReduces", "type": "int",  "default": -1}
      ]
     },

     {"type": "record", "name": "MapAttemptFinished",
      "fields": [
          {"name": "taskid", "type": "string"},
          {"name": "attemptId", "type": "string"},
          {"name": "taskType", "type": "string"},
          {"name": "taskStatus", "type": "string"},
          {"name": "mapFinishTime", "type": "long"},
          {"name": "finishTime", "type": "long"},
          {"name": "hostname", "type": "string"},
          {"name": "port", "type": "int"},
          {"name": "rackname", "type": "string"},
          {"name": "state", "type": "string"},
          {"name": "counters", "type": "JhCounters"},
          {"name": "clockSplits", "type": { "type": "array", "items": "int"}},
          {"name": "cpuUsages", "type": { "type": "array", "items": "int"}},
          {"name": "vMemKbytes", "type": { "type": "array", "items": "int"}},
          {"name": "physMemKbytes", "type": { "type": "array", "items": "int"}}
      ]
     },

     {"type": "record", "name": "ReduceAttemptFinished",
      "fields": [
          {"name": "taskid", "type": "string"},
          {"name": "attemptId", "type": "string"},
          {"name": "taskType", "type": "string"},
          {"name": "taskStatus", "type": "string"},
          {"name": "shuffleFinishTime", "type": "long"},
          {"name": "sortFinishTime", "type": "long"},
          {"name": "finishTime", "type": "long"},
          {"name": "hostname", "type": "string"},
          {"name": "port", "type": "int"},
          {"name": "rackname", "type": "string"},
          {"name": "state", "type": "string"},
          {"name": "counters", "type": "JhCounters"},
          {"name": "clockSplits", "type": { "type": "array", "items": "int"}},
          {"name": "cpuUsages", "type": { "type": "array", "items": "int"}},
          {"name": "vMemKbytes", "type": { "type": "array", "items": "int"}},
          {"name": "physMemKbytes", "type": { "type": "array", "items": "int"}}
      ]
     },

     {"type": "record", "name": "TaskAttemptFinished",
      "fields": [
          {"name": "taskid", "type": "string"},
          {"name": "attemptId", "type": "string"},
          {"name": "taskType", "type": "string"},
          {"name": "taskStatus", "type": "string"},
          {"name": "finishTime", "type": "long"},
          {"name": "rackname", "type": "string"},
          {"name": "hostname", "type": "string"},
          {"name": "state", "type": "string"},
          {"name": "counters", "type": "JhCounters"}
      ]
     },

     {"type": "record", "name": "TaskAttemptStarted",
      "fields": [
          {"name": "taskid", "type": "string"},
          {"name": "taskType", "type": "string"},
          {"name": "attemptId", "type": "string"},
          {"name": "startTime", "type": "long"},
          {"name": "trackerName", "type": "string"},
          {"name": "httpPort", "type": "int"},
          {"name": "shufflePort", "type": "int"},
          {"name": "containerId", "type": "string"},
          {"name": "locality", "type": ["null","string"], "default": null},
          {"name": "avataar", "type": ["null","string"], "default": null}
      ]
     },

     {"type": "record", "name": "TaskAttemptUnsuccessfulCompletion",
      "fields": [
          {"name": "taskid", "type": "string"},
          {"name": "taskType", "type": "string"},
          {"name": "attemptId", "type": "string"},
          {"name": "finishTime", "type": "long"},
          {"name": "hostname", "type": "string"},
          {"name": "port", "type": "int"},
          {"name": "rackname", "type": "string"},
          {"name": "status", "type": "string"},
          {"name": "error", "type": "string"},
          {"name": "counters", "type": ["null","JhCounters"], "default": null},
          {"name": "clockSplits", "type": { "type": "array", "items": "int"}},
          {"name": "cpuUsages", "type": { "type": "array", "items": "int"}},
          {"name": "vMemKbytes", "type": { "type": "array", "items": "int"}},
          {"name": "physMemKbytes", "type": { "type": "array", "items": "int"}}
      ]
     },

     {"type": "record", "name": "TaskFailed",
      "fields": [
          {"name": "taskid", "type": "string"},
          {"name": "taskType", "type": "string"},
          {"name": "finishTime", "type": "long"},
          {"name": "error", "type": "string"},
          {"name": "failedDueToAttempt", "type": ["null", "string"] },
          {"name": "status", "type": "string"},
          {"name": "counters", "type": ["null","JhCounters"], "default": null}
      ]
     },

     {"type": "record", "name": "TaskFinished",
      "fields": [
          {"name": "taskid", "type": "string"},
          {"name": "taskType", "type": "string"},
          {"name": "finishTime", "type": "long"},
          {"name": "status", "type": "string"},
          {"name": "counters", "type": "JhCounters"},
          {"name": "successfulAttemptId", "type": ["null","string"], "default": null}
      ]
     },
     	
     {"type": "record", "name": "TaskStarted",
      "fields": [
          {"name": "taskid", "type": "string"},
          {"name": "taskType", "type": "string"},
          {"name": "startTime", "type": "long"},
          {"name": "splitLocations", "type": "string"}
      ]
     },

     {"type": "record", "name": "TaskUpdated",
      "fields": [
          {"name": "taskid", "type": "string"},
          {"name": "finishTime", "type": "long"}
      ]
     },

     {"type": "enum", "name": "EventType",
      "symbols": [
          "JOB_SUBMITTED",
          "JOB_INITED",
          "JOB_FINISHED",
          "JOB_PRIORITY_CHANGED",
          "JOB_STATUS_CHANGED",
          "JOB_QUEUE_CHANGED",
          "JOB_FAILED",
          "JOB_KILLED",
          "JOB_ERROR",
          "JOB_INFO_CHANGED",
          "TASK_STARTED",
          "TASK_FINISHED",
          "TASK_FAILED",
          "TASK_UPDATED",
          "NORMALIZED_RESOURCE",
          "MAP_ATTEMPT_STARTED",
          "MAP_ATTEMPT_FINISHED",
          "MAP_ATTEMPT_FAILED",
          "MAP_ATTEMPT_KILLED",
          "REDUCE_ATTEMPT_STARTED",
          "REDUCE_ATTEMPT_FINISHED",
          "REDUCE_ATTEMPT_FAILED",
          "REDUCE_ATTEMPT_KILLED",
          "SETUP_ATTEMPT_STARTED",
          "SETUP_ATTEMPT_FINISHED",
          "SETUP_ATTEMPT_FAILED",
          "SETUP_ATTEMPT_KILLED",
          "CLEANUP_ATTEMPT_STARTED",
          "CLEANUP_ATTEMPT_FINISHED",
          "CLEANUP_ATTEMPT_FAILED",
          "CLEANUP_ATTEMPT_KILLED",
          "AM_STARTED"
          ]
     },

     {"type": "record", "name": "Event",
      "fields": [
          {"name": "type", "type": "EventType"},
          {"name": "event",
           "type": [
               "JobFinished",
               "JobInfoChange",
               "JobInited",
               "AMStarted",
               "JobPriorityChange",
               "JobQueueChange",
               "JobStatusChanged",
               "JobSubmitted",
               "JobUnsuccessfulCompletion",
               "MapAttemptFinished",
               "ReduceAttemptFinished",
               "TaskAttemptFinished",
               "TaskAttemptStarted",
               "TaskAttemptUnsuccessfulCompletion",
               "TaskFailed",
               "TaskFinished",
               "TaskStarted",
               "TaskUpdated"
               ]
          }
      ]
     }

 ],

 "messages": {}
}
```

## 3. MapReduce目录详解

mapreduce目录包含开发MapReduce程序所需要的mr客户端类源码。

## 4. client-core模块源码分析时间序列

重点对其中的类进行分析。从Driver驱动器程序配置Job，到提交Job，读取输入数据，执行数据处理，输出数据整个流程串行对源码分析。后续可以根据下表循序渐进分析源码

| 类名                                                        | 所属模块                          | 剖析日期   |
| ----------------------------------------------------------- | --------------------------------- | ---------- |
| org.apache.hadoop.mapreduce.Job                             | hadoop-mapreduce-client-core      | 2020/04/26 |
| org.apache.hadoop.mapreduce.JobSubmitter                    | hadoop-mapreduce-client-core      | 2020/04/27 |
| org.apache.hadoop.mapreduce.Cluster                         | hadoop-mapreduce-client-core      | 2020/04/28 |
| org.apache.hadoop.mapreduce.protocol.ClientProtocolProvider | hadoop-mapreduce-client-core      | 2020/04/28 |
| org.apache.hadoop.mapred.YARNRunner                         | hadoop-mapreduce-client-jobclient | 2020/04/29 |

