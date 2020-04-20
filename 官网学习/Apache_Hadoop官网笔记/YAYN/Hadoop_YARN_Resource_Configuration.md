# Hadoop: YARN Resource Configuration

## 1. 概述

YARN支持可扩展的资源模型。 默认情况下，YARN跟踪所有节点，应用程序和队列的CPU和内存，但是资源定义可以扩展为包括任意可计数资源。可计数资源是在容器运行时消耗但随后释放的资源。 CPU和内存都是可数的资源。 其他包括GPU资源和软件许可证。 

此外，YARN还支持使用资源配置文件，这使用户可以通过单个配置文件指定多个资源请求，类似于Amazon Web Services Elastic Compute Cluster实例类型。 例如，“大”可能意味着8个虚拟内核和16GB RAM。

## 2. 配置

支持以下配置属性。

**yarn-site.xml**

| Configuration Property                           | Description                          |
| :----------------------------------------------- | :----------------------------------- |
| `yarn.resourcemanager.resource-profiles.enabled` | 是否启用资源配置文件。默认为`false`. |

**resource-types.xml**

| Configuration Property                    | Description                                               |
| :---------------------------------------- | :-------------------------------------------------------- |
| `yarn.resource-types`                     | 逗号分割的资源列表，包括：`memory`, `memory-mb`, `vcores` |
| `yarn.resource-types..units`              | 指定资源类型的默认单位                                    |
| `yarn.resource-types..minimum-allocation` | 指定资源类型的最小请求量                                  |
| `yarn.resource-types..maximum-allocation` | 指定资源类型的最大请求量                                  |

**node-resources.xml**

| Configuration Property                       | Description                       |
| :------------------------------------------- | :-------------------------------- |
| `yarn.nodemanager.resource-type.`< resource> | NodeManager中可用的指定资源的数量 |

注意，如果使用resource-types.xml和node-resources.xml文件，它们也需要与yarn-site.xml放在同一配置目录中。 另外，也可以将它们中的属性放置在yarn-site.xml文件中。

## 3. YARN资源模型

### 3.1 Resource Manager

资源管理器是跟踪集群中哪些资源的最终决定者， 资源管理器从XML配置文件加载其资源定义。例如：定义除CPU、内存之外的新资源，应该配置以下属性：

```xml
<configuration>
  <property>
    <name>yarn.resource-types</name>
    <value>resource1,resource2</value>
    <description>
    The resources to be used for scheduling. Use resource-types.xml
    to specify details about the individual resource types.
    </description>
  </property>
</configuration>
```

有效的资源名称必须以字母开头，并且只能包含字母，数字和以下任意一个：‘.’，‘_’，或者 ‘-’。有效的资源名称还可以在名称空间之后加上斜杠。 有效的名称空间由句点分隔的字母，数字和破折号组成。 例如，以下是有效的资源名称：

- myresource
- my_resource
- My-Resource01
- com.acme/myresource

以下是非法的资源名称示例：

- 10myresource
- my resource
- com/acme/myresource
- $NS/myresource
- -none-/myresource

对于定义的每种新资源类型，可以添加可选的单位属性以设置资源类型的默认单位。 有效值为：

| Unit Name | Meaning                  |
| :-------- | :----------------------- |
| p         | pico                     |
| n         | nano                     |
| u         | micro                    |
| m         | milli                    |
|           | default,  no unit        |
| k         | kilo                     |
| M         | mega                     |
| G         | giga                     |
| T         | tera                     |
| P         | peta                     |
| Ki        | binary kilo, i.e. 1024   |
| Mi        | binary mega, i.e. 1024^2 |
| Gi        | binary giga, i.e. 1024^3 |
| Ti        | binary tera, i.e. 1024^4 |
| Pi        | binary peta, i.e. 1024^5 |

该属性必须命名为`yarn.resource-types.<resource>.units`。每个定义的资源还可以具有可选的最小值和最大值属性。 这些属性必须分别命名为`yarn.resource-types.<resource>.minimum-allocation`和`yarn.resource-types.<resource>.maximum-allocation`。

yarn.resource-types属性以及unit、minimum、maximum，通常在yarn-site.xml或resource-types.xml中定义。 例如，以下内容可能会出现在两个文件中：

```xml
<configuration>
  <property>
    <name>yarn.resource-types</name>
    <value>resource1, resource2</value>
  </property>

  <property>
    <name>yarn.resource-types.resource1.units</name>
    <value>G</value>
  </property>

  <property>
    <name>yarn.resource-types.resource2.minimum-allocation</name>
    <value>1</value>
  </property>

  <property>
    <name>yarn.resource-types.resource2.maximum-allocation</name>
    <value>1024</value>
  </property>
</configuration>
```

### 3.2 Node Manager

每个节点管理器独立定义该节点可用的资源。 通过为每个可用资源设置属性来完成资源定义。 该属性必须命名为`yarn.nodemanager.resource-type.<resource>`，并且可以放置在yarn-site.xml或node­resources.xml。该属性的值应为节点提供的资源量。 例如：

```xml
<configuration>
 <property>
   <name>yarn.nodemanager.resource-type.resource1</name>
   <value>5G</value>
 </property>

 <property>
   <name>yarn.nodemanager.resource-type.resource2</name>
   <value>2m</value>
 </property>

</configuration>
```

注意，nodeManager中资源的单位不必与资源管理器所保存的定义相匹配。 如果单位不匹配，资源管理器将自动进行转换。

### 3. MapReduce使用资源

MapReduce向资源管理器请求三种容器类型：application master容器、map容器和reduce容器。对于每种容器类型，都有一组对应的属性用于设置请求的资源。

用于在MapReduce中设置资源请求的属性为：

| Property                                    | Description                                                  |
| :------------------------------------------ | :----------------------------------------------------------- |
| `yarn.app.mapreduce.am.resource.mb`         | Sets the memory requested for the application master container to the value in MB. No longer preferred. Use `yarn.app.mapreduce.am.resource.memory-mb` instead. Defaults to 1536. |
| `yarn.app.mapreduce.am.resource.memory`     | Sets the memory requested for the application master container to the value in MB. No longer preferred. Use `yarn.app.mapreduce.am.resource.memory-mb` instead. Defaults to 1536. |
| `yarn.app.mapreduce.am.resource.memory-mb`  | Sets the memory requested for the application master container to the value in MB. Defaults to 1536. |
| `yarn.app.mapreduce.am.resource.cpu-vcores` | Sets the CPU requested for the application master container to the value. No longer preferred. Use `yarn.app.mapreduce.am.resource.vcores` instead. Defaults to 1. |
| `yarn.app.mapreduce.am.resource.vcores`     | Sets the CPU requested for the application master container to the value. Defaults to 1. |
| `yarn.app.mapreduce.am.resource.`           | Sets the quantity requested of `` for the application master container to the value. If no unit is specified, the default unit for the resource is assumed. See the section on units above. |
| `mapreduce.map.memory.mb`                   | Sets the memory requested for the all map task containers to the value in MB. No longer preferred. Use `mapreduce.map.resource.memory-mb` instead. Defaults to 1024. |
| `mapreduce.map.resource.memory`             | Sets the memory requested for the all map task containers to the value in MB. No longer preferred. Use `mapreduce.map.resource.memory-mb` instead. Defaults to 1024. |
| `mapreduce.map.resource.memory-mb`          | Sets the memory requested for the all map task containers to the value in MB. Defaults to 1024. |
| `mapreduce.map.cpu.vcores`                  | Sets the CPU requested for the all map task containers to the value. No longer preferred. Use `mapreduce.map.resource.vcores` instead. Defaults to 1. |
| `mapreduce.map.resource.vcores`             | Sets the CPU requested for the all map task containers to the value. Defaults to 1. |
| `mapreduce.map.resource.`                   | Sets the quantity requested of `` for the all map task containers to the value. If no unit is specified, the default unit for the resource is assumed. See the section on units above. |
| `mapreduce.reduce.memory.mb`                | Sets the memory requested for the all reduce task containers to the value in MB. No longer preferred. Use `mapreduce.reduce.resource.memory-mb` instead. Defaults to 1024. |
| `mapreduce.reduce.resource.memory`          | Sets the memory requested for the all reduce task containers to the value in MB. No longer preferred. Use `mapreduce.reduce.resource.memory-mb` instead. Defaults to 1024. |
| `mapreduce.reduce.resource.memory-mb`       | Sets the memory requested for the all reduce task containers to the value in MB. Defaults to 1024. |
| `mapreduce.reduce.cpu.vcores`               | Sets the CPU requested for the all reduce task containers to the value. No longer preferred. Use `mapreduce.reduce.resource.vcores` instead. Defaults to 1. |
| `mapreduce.reduce.resource.vcores`          | Sets the CPU requested for the all reduce task containers to the value. Defaults to 1. |
| `mapreduce.reduce.resource.`                | Sets the quantity requested of `` for the all reduce task containers to the value. If no unit is specified, the default unit for the resource is assumed. See the section on units above. |

注意，YARN可以修改这些资源请求，以满足配置的最小和最大资源值，或者是配置的增量的倍数。 请参阅YARN调度器配置中的属性：yarn.scheduler.maximum-allocation-mb, yarn.scheduler.minimum-allocation-mb, yarn.scheduler.increment-allocation-mb, yarn.scheduler.maximum-allocation-vcores, yarn.scheduler.minimum-allocation-vcores, yarn.scheduler.increment-allocation-vcores

## 4. Resource Profiles

Resource Profile为用户提供了一种使用单个配置文件请求一组资源的简便方法，并为管理员提供了一种管理消耗资源的方式。

要配置资源类型，管理员必须在资源管理器的yarn-site.xml文件中将yarn.resourcemanager.resource-profiles.enabled设置为true。 该文件定义了受支持的配置文件。 例如：

```json
{
    "small": {
        "memory-mb" : 1024,
        "vcores" : 1
    },
    "default" : {
        "memory-mb" : 2048,
        "vcores" : 2
    },
    "large" : {
        "memory-mb": 4096,
        "vcores" : 4
    },
    "compute" : {
        "memory-mb" : 2048,
        "vcores" : 2,
        "gpu" : 1
    }
}
```

在此示例中，用户可以访问具有不同资源设置的四个配置文件。 请注意，在计算(compute)配置中，管理员已如上所述配置了其他资源(gpu)。

### 4.1 Requesting Profiles

distributed shell是当前唯一支持资源配置文件的客户端。 使用distributed shell，用户可以指定资源配置文件名称，该名称将自动转换为适当的一组资源请求：

```
hadoop job $DISTSHELL -jar $DISTSHELL -shell_command run.sh -container_resource_profile small
```

