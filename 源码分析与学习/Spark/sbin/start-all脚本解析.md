# Start-all.sh Spark启动脚本解析

## 1. 脚本作用

start-all.sh脚本位于Spark源码的sbin目录下，用于加载Spark配置并启动所有Spark守护进程(master，worker)。

## 2. 源码

```sh
# Start all spark daemons.
# Starts the master on this node.
# Starts a worker on each node specified in conf/slaves

# 如果没有在机器中配置SPARK_HOME环境变量
if [ -z "${SPARK_HOME}" ]; then
# 指定SPARK_HOME为 当前sbin目录的上一级目录
  export SPARK_HOME="$(cd "`dirname "$0"`"/..; pwd)"
fi

# Load the Spark configuration
. "${SPARK_HOME}/sbin/spark-config.sh"

# Start Master
"${SPARK_HOME}/sbin"/start-master.sh

# Start Workers
"${SPARK_HOME}/sbin"/start-slaves.sh
```

