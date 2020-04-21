# Hadoop Rack Awareness

hadoop组件是支持机架的。例如，HDFS块放置将通过在不同的机架上放置一个块副本来使用机架感知容错。这在网络交换机故障或集群中的分区时提供了数据可用性。

Hadoop master守护进程通过调用配置文件指定的外部脚本或java类来获取集群工作者的机架id。对于拓扑使用java类或外部脚本，输出必须遵循`java org.apache.hadoop.net.DNSToSwitchMapping`接口。该接口需要维护一对一的对应关系，拓扑信息的格式为“/myrack/myhost”，其中“/”是拓扑分隔符，“myrack”是机架标识符，“myhost”是单独的主机。假设每个机架有一个/ 24个子网，可以使用'/192.168.100.0/192.168.100.5'格式作为唯一的机架 - 主机拓扑映射。

要使用java类进行拓扑映射，类名由配置文件中的**net.topology.node.switch.mapping.impl**参数指定。例如，NetworkTopology.java包含在hadoop发行版中，可以由Hadoop管理员自定义。使用Java类而不是外部脚本具有性能优势，因为当新的工作节点注册自身时，Hadoop不需要派生外部进程。

如果实现外部脚本，将使用配置文件中的**net.topology.script.file.name**参数指定它。与java类不同，外部拓扑脚本不包含在Hadoop发行版中，由管理员提供。在分叉拓扑脚本时，Hadoop将向ARGV发送多个IP地址。发送到拓扑脚本的IP地址数由**net.topology.script.number.args**控制，默认为100.如果**net.topology.script.number.args**更改为1，则每个拓扑脚本都会分叉DataNodes和/或NodeManagers提交的IP。

**Python Example**

```python
#!/usr/bin/python
# this script makes assumptions about the physical environment.
#  1)每个机架是它自己的第3层网络与/24子网，
# 这可能是典型的每个机架有自己的交换机与上行到一个中心核心路由器。
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
# 2) topology script 表获取IP的作为输入，计算网络地址，并打印“/ network_address / IP”
import netaddr
import sys
 # discard name of topology script from argv list as we just want IP addresses
sys.argv.pop(0)                                                 

# set netmask(子码掩码) to what's being used in your environment.  The example uses a /24
netmask = '255.255.255.0'                                        

 # loop over list of datanode IP's
for ip in sys.argv:                                             
    # format address string so it looks like 'ip/netmask' to make netaddr work
    address = '{0}/{1}'.format(ip, netmask)                      
    try:
        network_address = netaddr.IPNetwork(address).network     
        # calculate and print network address
        print ("/{0}".format(network_address))
    except:
        # print catch-all value if unable to calculate network address
        print ("/rack-unknown")
```

**bach Example**

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
# With this network topology, we are treating each host as a rack.  This is being done by taking the last octet
# in the datanode's IP and prepending it with the word '/rack-'.  The advantage for doing this is so HDFS
# can create its 'off-rack' block copy.
# 1) 'echo $@' will echo all ARGV values to xargs.
# 2) 'xargs' will enforce that we print a single argv value per line
# 3) 'awk' will split fields on dots and append the last field to the string '/rack-'. If awk
#    fails to split on four dots, it will still print '/rack-' last field value

echo $@ | xargs -n 1 | awk -F '.' '{print "/rack-"$NF}'
```

