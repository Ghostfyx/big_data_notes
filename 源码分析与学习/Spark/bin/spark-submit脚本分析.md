# spark-submit 脚本源码分析

## 1. spark-submit的入口函数

Spark 官网Spark任务提交示例如下，调用spark-submit脚本提交Application到Spark集群。

```sh
$ ./bin/spark-submit --class org.apache.spark.examples.SparkPi \
    --master yarn \
    --deploy-mode cluster \
    --driver-memory 4g \
    --executor-memory 2g \
    --executor-cores 1 \
    --queue thequeue \
    examples/jars/spark-examples*.jar \
    10
```

## 2. spark-sbumit脚本

spark-submit脚本位于Spark项目的bin目录下

```sh
#!/usr/bin/env bash

#
# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#    http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#

if [ -z "${SPARK_HOME}" ]; then
  source "$(dirname "$0")"/find-spark-home
fi

# disable randomized hash for string in Python 3.3+
export PYTHONHASHSEED=0

# 调用SparkSubmit类，执行提交逻辑 "$@"表示获取所有用户提交Application传入的参数
exec "${SPARK_HOME}"/bin/spark-class org.apache.spark.deploy.SparkSubmit "$@"
```

## 3. Spark-submit与Spark-start-all区别

Spark on YARN模式下，不需要启动Master、Worker守护进程调度资源完成Driver-Exceutor-Task计算框架的计算，因此只需要使用Spark-submit脚本提交应用，由YARN的RM与NM完成各个节点的资源管理与分配，Spark在YARN RM中申请AM(Driver)作为Spark作业的管理器，AM负责管理/调度Executor与Task。