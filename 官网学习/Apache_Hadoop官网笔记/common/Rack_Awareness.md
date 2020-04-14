# Rack Awareness

Hadoop组件可识别机架。 例如，通过将一个块副本放置在不同的机架上，HDFS块放置将利用机架感知来实现容错。 在网络交换机发生故障或集群中存在分区时，将提供数据高可用性。

Hadoop主守护程序通过调用配置文件指定的外部脚本或Java类来获取集群worker的机架ID。 使用Java类或外部脚本进行网络拓扑构建，其输出必须遵守java org.apache.hadoop.net.DNSToSwitchMapping接口。接口维护一对一的对应关系，并且拓扑信息的格式为`/myrack/myhost`，其中“/”为拓扑分隔符，myrack为机架标识符，myhost为机器主机名。假设每个机架有一个/24子网，则可以使用`/192.168.100.0/192.168.100.5`的格式作为唯一的机架主机拓扑映射。

由配置文件中的net.topology.node.switch.mapping.impl参数指用于拓扑映射的Java类。hadoop分发中包含一个示例NetworkTopology.java，可以由Hadoop管理员自定义。 使用Java类而不是外部脚本具有性能优势，因为当新的workNode向namenode注册时，Hadoop不需要派生外部进程。

如果实现外部脚本，则要在配置文件中使用net.topology.script.file.name参数指定。与java类不同，Hadoop分发版不包含外部拓扑脚本，而是由管理员提供的。生成拓扑脚本时，Hadoop将向ARGV发送多个IP地址。 发送到拓扑脚本的IP地址数量由net.topology.script.number.args控制，默认100。如果更改为1，对于datanode和/或nodemanager提交的每个IP，都会生成一个拓扑脚本。

如果未设置net.topology.script.file.name或net.topology.node.switch.mapping.impl，则为任何传递的IP地址返回机架ID“/default-rack”。这样会导致HDFS块复制异常，因为默认是在多个机架上写入一个复制的块，由于只有一个名为“/default-rack”的机架导致无法这样做。

## Python网络拓扑脚本

```python
#!/usr/bin/python
# this script makes assumptions about the physical environment.
#  1) each rack is its own layer 3 network with a /24 subnet, which
# could be typical where each rack has its own
#     switch with uplinks to a central core router.
#
#             +-----------+
#             |core router|
#             +-----------+
#            /             \
#   +-----------+        +-----------+
#   |rack switch|        |rack switch|
#   +-----------+        +-----------+
#   | data node |        | data node |
#   +-----------+        +-----------+
#   | data node |        | data node |
#   +-----------+        +-----------+
#
# 2) topology script gets list of IP's as input, calculates network address, and prints '/network_address/ip'.

import netaddr
import sys
sys.argv.pop(0) #discard name of topology script from argv list as we just want IP addresses

netmask = '255.255.255.0' #set netmask to what's being used in your environment.The example uses a /24
for ip in sys.argv:     # loop over list of datanode IP's
    address = '{0}/{1}'.format(ip,netmask)# format address string so it looks like 'ip/netmask' to make netaddr work
    try:
        network_address = netaddr.IPNetwork(address).network # calculate and print network address
        print "/{0}".format(network_address)
    except:
        print "/rack-unknown" # print catch-all value if unable to calculate network address
```

## Bash网络拓扑脚本

```bash
#!/usr/bin/env bash
# Here's a bash example to show just how simple these scripts can be
# Assuming we have flat network with everything on a single switch, we can fake a rack topology.
# This could occur in a lab environment where we have limited nodes,like 2-8 physical machines on a unmanaged switch.
# This may also apply to multiple virtual machines running on the same physical hardware.
# The number of machines isn't important, but that we are trying to fake a network topology when there isn't one.
#
#       +----------+    +--------+
#       |jobtracker|    |datanode|
#       +----------+    +--------+
#              \        /
#  +--------+  +--------+  +--------+
#  |datanode|--| switch |--|datanode|
#  +--------+  +--------+  +--------+
#              /        \
#       +--------+    +--------+
#       |datanode|    |namenode|
#       +--------+    +--------+
#
# With this network topology, we are treating each host as a rack. This is being done by taking the last octet
# in the datanode's IP and prepending it with the word '/rack-'.  The advantage for doing this is so HDFS
# can create its 'off-rack' block copy.
# 1) 'echo $@' will echo all ARGV values to xargs.
# 2) 'xargs' will enforce that we print a single argv value per line
# 3) 'awk' will split fields on dots and append the last field to the string '/rack-'. If awk
# fails to split on four dots, it will still print '/rack-' last field value

echo $@ | xargs -n 1 | awk -F '.' '{print "/rack-"$NF}'
```

