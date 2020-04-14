# linux下Elasticsearch 安装、配置及示例

## 1. Elasticsearch关键名词

| 名词     | 描述                                                         |
| -------- | ------------------------------------------------------------ |
| index    | 相当于一个数据库                                             |
| type     | 相当于数据库里的一个表                                       |
| id       | 唯一，相当于主键                                             |
| node     | 节点是es实例，一台机器可以运行多个实例，但是同一台机器上的实例在配置文件中要确保http和tcp端口不同 |
| cluster  | 代表一个集群，集群中有多个节点，其中有个节点会被选为主节点，这个主节点是可以通过选举产生的，主从节点是对于集群内部来说的。 |
| shards   | 代表索引分片，ES可以把一个完整的索引分成多个分片，这样的好处是可以把一个大的索引拆分成多个，分布到不同的节点上，构成分布式搜索。分片的数量只能在索引创建前指定，并且索引创建后不能更改。 |
| replicas | 代表索引的复本，es可以设置多个索引的复本，副本的作用一是提高系统的容错性，当个某个节点某个分片损坏或丢失时可以从副本中恢复。二是提高es的查询效率，es会自动对搜索请求进行负载均衡。 |

## 2. Elasticsearch 下载

由于高版本elasticsearch需要高版本JDK支持，因此使用5.6.16版本+JDK8，Linux下载地址：

```
wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-5.6.16.tar.gz
```

解压安装包：

```
tar -zxvf elasticsearch-5.6.16.tar.gz
```

## 3, 修改配置文件

注意要安装低版本Elasticsearch，Elasticsearch内置JDK，elasticsearch-7.6.2需要JDK11的支持。因此卸载elasticsearch-7.6.2，安装elasticsearch-5.6.16。修改/conf/elasticsearch.yml

```yaml
cluster.name: my_elasticsearch1
node.name: elasticsearch1_node1
path.data: /home/hadoop/elasticsearch/data
path.logs: /home/hadoop/elasticsearch/logs
bootstrap.memory_lock: false
network.host: 192.168.234.101
http.port: 9200
```

## 4. 安装遇到错误

### 4.1 

```
[2018-05-18T17:44:59,658][INFO ][o.e.b.BootstrapChecks    ] [gFOuNlS] bound or publishing to a non-loopback address, enforcing bootstrap checks
ERROR: [2] bootstrap checks failed
[1]: max file descriptors [4096] for elasticsearch process is too low, increase to at least [65536]
[2]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
[1]: max file descriptors [65535] for elasticsearch process is too low, increase to at least [65536]
```

编辑 /etc/security/limits.conf，追加以下内容；

```
* soft nofile 300000
* hard nofile 300000
* soft nproc 102400
* soft memlock unlimited
* hard memlock unlimited
```

### 4.2 

```
[2]: max virtual memory areas vm.max_map_count [65530] is too low, increase to at least [262144]
```

修复方法：

```sh
echo "vm.max_map_count=262144" > /etc/sysctl.conf
sysctl -p
```

