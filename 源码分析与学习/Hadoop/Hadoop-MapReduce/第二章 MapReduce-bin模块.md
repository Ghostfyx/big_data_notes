# 第二章 bin模块

## 1. 概述

Hadoop MapReduce计算框架命令行使用详情以及命令解析。具有三种文件类型：.sh文件、.cmd文件、.bash脚本文件，支持不同的操作系统。

## 2. 脚本文件详解

bin模块具有以下文件：

- mapred bash脚本文件
- mapred.cmd  
- mapred-config.cmd
- mapred-config.sh
- mr-jobhistory-daemon.sh

### 2.1 mapred脚本详解

所有的mapreduce命令都由`bin/mapred`脚本调用。

```bash
# 正在执行的脚本的名称
HADOOP_SHELL_EXECNAME="mapred"
# 获取当前执行的脚本文件的全路径，等价于与${BASH_SOURCE}
MYNAME="${BASH_SOURCE-$0}"

## @description  build up the mapred command's usage text.
## @audience     public
## @stability    stable
## @replaceable  no
## 建立mapred命令的用法文本
function hadoop_usage
{
  hadoop_add_subcommand "classpath" client "prints the class path needed for running mapreduce subcommands"
  hadoop_add_subcommand "envvars" client "display computed Hadoop environment variables"
  hadoop_add_subcommand "historyserver" daemon "run job history servers as a standalone daemon"
  hadoop_add_subcommand "hsadmin" admin "job history server admin interface"
  hadoop_add_subcommand "job" client "manipulate MapReduce jobs"
  hadoop_add_subcommand "pipes" client "run a Pipes job"
  hadoop_add_subcommand "queue" client "get information regarding JobQueues"
  hadoop_add_subcommand "sampler" client "sampler"
  hadoop_add_subcommand "frameworkuploader" admin "mapreduce framework upload"
  hadoop_add_subcommand "version" client "print the version"
  hadoop_add_subcommand "minicluster" client "CLI MiniCluster"
  hadoop_generate_usage "${HADOOP_SHELL_EXECNAME}" true
}
```

 运行不带任何参数的mapred脚本将打印所有命令的描述，即hadoop_usage作用是输出mapred命令的描述。运行示例如下：

```
Usage: mapred [--config confdir] COMMAND
       where COMMAND is one of:
  pipes                run a Pipes job
  job                  manipulate MapReduce jobs
  queue                get information regarding JobQueues
  classpath            prints the class path needed for running
                       mapreduce subcommands
  historyserver        run job history servers as a standalone daemon
  distcp <srcurl> <desturl> copy file or directories recursively
  archive -archiveName NAME -p <parent path> <src>* <dest> create a hadoop archive
  archive-logs         combine aggregated logs into hadoop archives
  hsadmin              job history server admin interface

```

```bash
## @description  Default command handler for hadoop command
## @audience     public
## @stability    stable
## @replaceable  no
## @param        CLI arguments
function mapredcmd_case
{
  # 获取子命令
  subcmd=$1
  # shift命令用于对参数的移动(左移)，通常用于在不知道传入参数个数的情况下依次遍历每个参数然后进行相应处理
  shift
  
  # bash case循环命令语法
  case ${subcmd} in
    mradmin|jobtracker|tasktracker|groups)
      hadoop_error "Sorry, the ${subcmd} command is no longer supported."
      hadoop_error "You may find similar functionality with the \"yarn\" shell command."
      hadoop_exit_with_usage 1
    ;;
    classpath)
      hadoop_do_classpath_subcommand HADOOP_CLASSNAME "$@"
    ;;
    envvars)
      echo "JAVA_HOME='${JAVA_HOME}'"
      echo "HADOOP_MAPRED_HOME='${HADOOP_MAPRED_HOME}'"
      echo "MAPRED_DIR='${MAPRED_DIR}'"
      echo "MAPRED_LIB_JARS_DIR='${MAPRED_LIB_JARS_DIR}'"
      echo "HADOOP_CONF_DIR='${HADOOP_CONF_DIR}'"
      echo "HADOOP_TOOLS_HOME='${HADOOP_TOOLS_HOME}'"
      echo "HADOOP_TOOLS_DIR='${HADOOP_TOOLS_DIR}'"
      echo "HADOOP_TOOLS_LIB_JARS_DIR='${HADOOP_TOOLS_LIB_JARS_DIR}'"
      exit 0
    ;;
    historyserver)
      HADOOP_SUBCMD_SUPPORTDAEMONIZATION="true"
      HADOOP_CLASSNAME=org.apache.hadoop.mapreduce.v2.hs.JobHistoryServer
      if [[ -n "${HADOOP_JOB_HISTORYSERVER_HEAPSIZE}" ]]; then
        HADOOP_HEAPSIZE_MAX="${HADOOP_JOB_HISTORYSERVER_HEAPSIZE}"
      fi
      HADOOP_DAEMON_ROOT_LOGGER=${HADOOP_JHS_LOGGER:-$HADOOP_DAEMON_ROOT_LOGGER}
      if [[  "${HADOOP_DAEMON_MODE}" != "default" ]]; then
        hadoop_add_param HADOOP_OPTS mapred.jobsummary.logger "-Dmapred.jobsummary.logger=${HADOOP_DAEMON_ROOT_LOGGER}"
      fi
    ;;
    hsadmin)
      HADOOP_CLASSNAME=org.apache.hadoop.mapreduce.v2.hs.client.HSAdmin
    ;;
    job)
      HADOOP_CLASSNAME=org.apache.hadoop.mapred.JobClient
    ;;
    pipes)
      HADOOP_CLASSNAME=org.apache.hadoop.mapred.pipes.Submitter
    ;;
    queue)
      HADOOP_CLASSNAME=org.apache.hadoop.mapred.JobQueueClient
    ;;
    sampler)
      HADOOP_CLASSNAME=org.apache.hadoop.mapred.lib.InputSampler
    ;;
    frameworkuploader)
      HADOOP_CLASSNAME=org.apache.hadoop.mapred.uploader.FrameworkUploader
    ;;
    version)
      HADOOP_CLASSNAME=org.apache.hadoop.util.VersionInfo
    ;;
    minicluster)
      hadoop_add_classpath "${HADOOP_YARN_HOME}/${YARN_DIR}/timelineservice"'/*'
      hadoop_add_classpath "${HADOOP_YARN_HOME}/${YARN_DIR}/test"'/*'
      HADOOP_CLASSNAME=org.apache.hadoop.mapreduce.MiniHadoopClusterManager
    ;;
    *)
      HADOOP_CLASSNAME="${subcmd}"
      if ! hadoop_validate_classname "${HADOOP_CLASSNAME}"; then
        hadoop_exit_with_usage 1
      fi
    ;;
  esac
}
```

hadoop命令的默认命令处理程序。

```bash
bin=$(cd -P -- "$(dirname -- "${MYNAME}")" >/dev/null && pwd -P)

# let's locate libexec...
if [[ -n "${HADOOP_HOME}" ]]; then
  HADOOP_DEFAULT_LIBEXEC_DIR="${HADOOP_HOME}/libexec"
else
  HADOOP_DEFAULT_LIBEXEC_DIR="${bin}/../libexec"
fi

HADOOP_LIBEXEC_DIR="${HADOOP_LIBEXEC_DIR:-$HADOOP_DEFAULT_LIBEXEC_DIR}"
HADOOP_NEW_CONFIG=true
if [[ -f "${HADOOP_LIBEXEC_DIR}/mapred-config.sh" ]]; then
  # shellcheck source=./hadoop-mapreduce-project/bin/mapred-config.sh
  . "${HADOOP_LIBEXEC_DIR}/mapred-config.sh"
else
  echo "ERROR: Cannot execute ${HADOOP_LIBEXEC_DIR}/mapred-config.sh." 2>&1
  exit 1
fi

# now that we have support code, let's abs MYNAME so we can use it later
MYNAME=$(hadoop_abs "${MYNAME}")

if [ $# = 0 ]; then
  hadoop_exit_with_usage 1
fi

HADOOP_SUBCMD=$1
shift

if hadoop_need_reexec mapred "${HADOOP_SUBCMD}"; then
  hadoop_uservar_su mapred "${HADOOP_SUBCMD}" \
    "${MYNAME}" \
    "--reexec" \
    "${HADOOP_USER_PARAMS[@]}"
  exit $?
fi

hadoop_verify_user_perm "${HADOOP_SHELL_EXECNAME}" "${HADOOP_SUBCMD}"

HADOOP_SUBCMD_ARGS=("$@")

if declare -f mapred_subcommand_"${HADOOP_SUBCMD}" >/dev/null 2>&1; then
  hadoop_debug "Calling dynamically: mapred_subcommand_${HADOOP_SUBCMD} ${HADOOP_SUBCMD_ARGS[*]}"
  "mapred_subcommand_${HADOOP_SUBCMD}" "${HADOOP_SUBCMD_ARGS[@]}"
else
  mapredcmd_case "${HADOOP_SUBCMD}" "${HADOOP_SUBCMD_ARGS[@]}"
fi

hadoop_add_client_opts

if [[ ${HADOOP_WORKER_MODE} = true ]]; then
  hadoop_common_worker_mode_execute "${HADOOP_MAPRED_HOME}/bin/mapred" "${HADOOP_USER_PARAMS[@]}"
  exit $?
fi

hadoop_subcommand_opts "${HADOOP_SHELL_EXECNAME}" "${HADOOP_SUBCMD}"

# everything is in globals at this point, so call the generic handler
hadoop_generic_java_subcmd_handler
```

## 2.2 mapred-config 脚本详解

```bash
function hadoop_subproject_init
{
  if [[ -z "${HADOOP_MAPRED_ENV_PROCESSED}" ]]; then
    if [[ -e "${HADOOP_CONF_DIR}/mapred-env.sh" ]]; then
      . "${HADOOP_CONF_DIR}/mapred-env.sh"
      export HADOOP_MAPRED_ENV_PROCESSED=true
    fi
  fi

  # at some point in time, someone thought it would be a good idea to
  # create separate vars for every subproject.  *sigh*
  # let's perform some overrides and setup some defaults for bw compat
  # this way the common hadoop var's == subproject vars and can be
  # used interchangeable from here on out
  # ...
  # this should get deprecated at some point.
  
  # hadoop_deprecate_envvar将旧的配置替换为新的配置，在旧配置存在的情况下
  hadoop_deprecate_envvar HADOOP_MAPRED_LOG_DIR HADOOP_LOG_DIR

  hadoop_deprecate_envvar HADOOP_MAPRED_LOGFILE HADOOP_LOGFILE

  hadoop_deprecate_envvar HADOOP_MAPRED_NICENESS HADOOP_NICENESS

  hadoop_deprecate_envvar HADOOP_MAPRED_STOP_TIMEOUT HADOOP_STOP_TIMEOUT

  hadoop_deprecate_envvar HADOOP_MAPRED_PID_DIR HADOOP_PID_DIR

  hadoop_deprecate_envvar HADOOP_MAPRED_ROOT_LOGGER HADOOP_ROOT_LOGGER

  hadoop_deprecate_envvar HADOOP_JOB_HISTORYSERVER_OPTS MAPRED_HISTORYSERVER_OPTS

  HADOOP_MAPRED_HOME="${HADOOP_MAPRED_HOME:-$HADOOP_HOME}"

  hadoop_deprecate_envvar HADOOP_MAPRED_IDENT_STRING HADOOP_IDENT_STRING
}
```

初始化Hadoop mapReduce相关配置

```bash
if [[ -z "${HADOOP_LIBEXEC_DIR}" ]]; then
  _mc_this="${BASH_SOURCE-$0}"
  HADOOP_LIBEXEC_DIR=$(cd -P -- "$(dirname -- "${_mc_this}")" >/dev/null && pwd -P)
fi

# shellcheck source=./hadoop-common-project/hadoop-common/src/main/bin/hadoop-config.sh
if [[ -n "${HADOOP_COMMON_HOME}" ]] &&
   [[ -e "${HADOOP_COMMON_HOME}/libexec/hadoop-config.sh" ]]; then
  . "${HADOOP_COMMON_HOME}/libexec/hadoop-config.sh"
elif [[ -e "${HADOOP_LIBEXEC_DIR}/hadoop-config.sh" ]]; then
  . "${HADOOP_LIBEXEC_DIR}/hadoop-config.sh"
elif [ -e "${HADOOP_HOME}/libexec/hadoop-config.sh" ]; then
  . "${HADOOP_HOME}/libexec/hadoop-config.sh"
else
  echo "ERROR: Hadoop common not found." 2>&1
  exit 1
fi
```

### 2.3 mr-jobhistory-daemon脚本

mapReduce历史任务守护进程脚本。

```bash
function hadoop_usage
{
  echo "Usage: mr-jobhistory-daemon.sh [--config confdir] (start|stop|status) <hadoop-command> <args...>"
}
```

调用mr-jobhistory-daemon.sh 脚本不带任何参数时，会执行hadoop_usage函数。

```bash
# let's locate libexec...
if [[ -n "${HADOOP_HOME}" ]]; then
  HADOOP_DEFAULT_LIBEXEC_DIR="${HADOOP_HOME}/libexec"
else
  this="${BASH_SOURCE-$0}"
  bin=$(cd -P -- "$(dirname -- "${this}")" >/dev/null && pwd -P)
  HADOOP_DEFAULT_LIBEXEC_DIR="${bin}/../libexec"
fi

HADOOP_LIBEXEC_DIR="${HADOOP_LIBEXEC_DIR:-$HADOOP_DEFAULT_LIBEXEC_DIR}"
# shellcheck disable=SC2034
HADOOP_NEW_CONFIG=true
if [[ -f "${HADOOP_LIBEXEC_DIR}/yarn-config.sh" ]]; then
  . "${HADOOP_LIBEXEC_DIR}/yarn-config.sh"
else
  echo "ERROR: Cannot execute ${HADOOP_LIBEXEC_DIR}/yarn-config.sh." 2>&1
  exit 1
fi

daemonmode=$1
shift

hadoop_error "WARNING: Use of this script to ${daemonmode} the MR JobHistory daemon is deprecated."
hadoop_error "WARNING: Attempting to execute replacement \"mapred --daemon ${daemonmode}\" instead."

exec "${HADOOP_MAPRED_HOME}/bin/mapred" \
--config "${HADOOP_CONF_DIR}" --daemon "${daemonmode}" "$@"
```

