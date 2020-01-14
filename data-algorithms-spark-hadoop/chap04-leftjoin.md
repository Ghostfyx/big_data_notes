# Chap04 left join

这一章介绍如何在Hadoop环境中实现Left Outer Join外连接，会使用三种方式实现：

- MapReduce/Hadoop解决方案：使用传统的Mapper和Reducer
- Spark使用JavaRDD.leftOuterJoin()
- Spark不使用JavaRDD.leftOuterJoin()

## 左外连接示例

给定两个关系型数据库表：左表Users和右表Transactions，如下：

```python
users(user_id, location_id)
trancations(trancation_id,production_id,user_id,quantity,amount)
```

令左表$T_1$与右表$T_2$关系如下：
$$
T_1=(K, T_1) \\
T_2 = (K, T_2)
$$
$T_1,T_2$的左外连接将包含左表$T_1$的所有记录，即使连接条件在右表$T_2$中没有找到任何匹配的记录。如果左表对应一条记录而右表对应记录数为0，则结果仍会返回一行，不过$T_2$对应各列为NULL。形式化表示如下：
$$
LeftOuterJoin(T_1,T_2,K)=\{(k,t_1,t_2) \quad where \quad k \in T_1.k \quad and \quad k \in T_2.k\} \cup \{(k,t,null) \quad where \quad k \}
$$
![](http://www.pianshen.com/images/68/e530ce69902a3d28ede3d299a89a8344.png)

## 示例查询

### 插叙1 查询已售商品及关联的用户位置

```sql
select production_id, location_id from transations left join users on users.user_id = transations.user_id
```

###  查询2 查找已售商品关联用户位置的数量

```sql
select production_id, count(location_id) from transations left join users on users.user_id = transations.user_id group by production_id
```

### 查询3 查找已售商品关联唯一用户数

```sql
select production_id, count(distinct location_id) from transations left join users on users.user_id = transations.user_id group by production_id 
```

## MapReduce 左外连接实现

MapReduce主要分为两个阶段实现：

1. 找出所有已售商品以及关联地址，可以使用SQL查询1完成；
2. 找出所有已售商品以及关联唯一地址/地址数量，可以使用上面SQL查询3实现

### MapReduce阶段1

具体步骤如下：

- 定义用户映射器(UsersMapper)和交易映射器(TransationsMapper)，利用**MultipleInputs**可以使用多个映射器；

	```java
	MultipleInputs.addInputPath(job, transactions, TextInputFormat.class, LeftJoinTransactionMapper.class);
	        MultipleInputs.addInputPath(job, users, TextInputFormat.class, LeftJoinUserMapper.class);
	```

	![](/Users/yuexiangfan/学习笔记/data-algorithms-spark-hadoop/img/chap04_leftjoin_1.jpeg)

- 自定义映射器分区函数与分组函数，分区函数用于将Mapper的输出数据映射到对应的Reduer，分组函数则将Reducer中的输入数据分组多次执行reduce函数。

- 定义Reducer函数，输出<producter_id><location_id>

### MapReduce 阶段2 统计唯一地址

将MapReduce阶段1的输出按照production_id分组，将location_id合并至Set集合，最终统计数量。

### Hadoop中的实现类

| 阶段  | 类名                         | 类描述                    |
| ----- | ---------------------------- | ------------------------- |
| 阶段1 | LeftJoinDriver               | 阶段1 Job驱动器           |
|       | LeftJoinReducer              | 左连接归约器              |
|       | LeftJoinTransactionMapper    | 左连接交易映射器          |
|       | LeftJoinUserMapper           | 左连接用户映射器          |
|       | SecondarySortPartitioner     | 左连接自然键分区类        |
|       | SecondarySortGroupComparator | 左连接Reducer自然键分组类 |
| 阶段2 | LocationCountDriver          | 阶段2 Job驱动器           |
|       | LocationCountMapper          | 定义Map完成地址统计       |
|       | LocationCountReducer         | 地址统计Reducer           |

### MapReducer代码，数据&运行脚本

见代码chap04

## Spark左外连接实现

与MapReduce API相比，Spark提供了更高层的API， 不再需要定义特殊的插件类，Spark RDD提供了不同类型的映射器(map()，flatmap( )，flatMapToPair()函数)，然后使用JavaRDD.union()函数返回Users和Transactions的JavaRDD的并集。union函数定义如下：

```java
JavaRDD<T> unionRDD = JavaRDD<T> this.union(JavaRDD<T> other)
JavaPariRDD<T> unionRDD = JavaPairRDD<T> this.union(JavaPairRDD<T> other)
```

**注意：**只能对相同类型的JavaRDD使用union函数

![](/Users/yuexiangfan/学习笔记/data-algorithms-spark-hadoop/img/chap04_leftjoin_2.jpeg)

### Spark程序的详细步骤

1. 读取User和Transaction数据文件，分别创建UsersJavaRDD和TransactionsJavaRDD；

2. 使用union函数合并UsersJavaRDD和TransactionsJavaRDD；

3. 调用RDD算子groupByKey()创建JavaPairRDD(userId, List<T_2>)

	```xml
	groupByKey的记录如下：
		<user_id, List[T2("L",location_id),T2("P",production_id1),T2("P",production_id2),T2("P",production_id3),T2("P",production_id4),...,T2("P",production_idn)]>
	```

4. 使用flatMapToPair函数创建productionLocationsRDD，数据格式如下：

	```xml
	(p1,location)
	(p2,location)
	(p3,location)
	(p4,location)
	...
	(pn,location)
	```

5. 使用groupByKey完成对商品的分组，并统计

### Spark实现代码

见SparkLeftOuterJoin类

## Spark使用leftOuterJoin()实现

