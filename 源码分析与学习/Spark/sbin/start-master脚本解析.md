# start-master 源码解析

## 1. 脚本作用

启动Master节点进程。

## 2. 脚本解析

```sh
# Starts the master on the machine this script is executed on.

# 读取/设置spark环境变量：SPARK_HOME
if [ -z "${SPARK_HOME}" ]; then
  export SPARK_HOME="$(cd "`dirname "$0"`"/..; pwd)"
fi

# NOTE: This exact class name is matched downstream by SparkSubmit.
# Any changes need to be reflected there.
# Master启动类的类名
CLASS="org.apache.spark.deploy.master.Master"

# $@用于shell脚本获取命名后的所有参数，如果是--help或者-h则打印帮助提示
if [[ "$@" = *--help ]] || [[ "$@" = *-h ]]; then
  echo "Usage: ./sbin/start-master.sh [options]"
  pattern="Usage:"
  pattern+="\|Using Spark's default log4j profile:"
  pattern+="\|Registered signal handlers for"

  "${SPARK_HOME}"/bin/spark-class $CLASS --help 2>&1 | grep -v "$pattern" 1>&2
  exit 1
fi

# master启动参数 = 用户自定义配置+Spark配置变量+Spark环境变量
ORIGINAL_ARGS="$@"
  # 加载Spark配置变量
. "${SPARK_HOME}/sbin/spark-config.sh"
  # 加载Spark环境变量
. "${SPARK_HOME}/bin/load-spark-env.sh"

# Spark master端口号
if [ "$SPARK_MASTER_PORT" = "" ]; then
  SPARK_MASTER_PORT=7077
fi

# 设置Spark master所在机器的hostname
if [ "$SPARK_MASTER_HOST" = "" ]; then
  case `uname` in
      (SunOS)
	  SPARK_MASTER_HOST="`/usr/sbin/check-hostname | awk '{print $NF}'`"
	  ;;
      (*)
	  SPARK_MASTER_HOST="`hostname -f`"
	  ;;
  esac
fi

# webUI端口号
if [ "$SPARK_MASTER_WEBUI_PORT" = "" ]; then
  SPARK_MASTER_WEBUI_PORT=8080
fi

# 核心！！！！启动最终执行的脚本
"${SPARK_HOME}/sbin"/spark-daemon.sh start $CLASS 1 \
  --host $SPARK_MASTER_HOST --port $SPARK_MASTER_PORT --webui-port $SPARK_MASTER_WEBUI_PORT \
  $ORIGINAL_ARGS
```

## 3. 脚本使用

### 3.1 帮助命令

```sh
./start-master.sh --help

Usage: ./sbin/start-master.sh [options]
D:\coding-env\Java\jdk1.8.0_71\bin\java -cp "F:\spark-2.4.0-bin-hadoop2.7/conf\;F:\spark-2.4.0-bin-hadoop2.7\jars\*" -Xmx1g org.apache.spark.deploy.master.Master --help
```

## 3.2 启动命令

```

```

