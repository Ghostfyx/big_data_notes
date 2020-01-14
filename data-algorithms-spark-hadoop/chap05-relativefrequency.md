# chap05-RelativeFrequency反转排序

本章重点介绍反转排序(Order Inversion, OI)设计模式，这种设计模式可以用来控制MapReduce框架中归约器值的顺序，通常会在数据分析阶段应用OI模式。在MapReduce与Spark中，值到达归约器(Reducer)的顺序是为定义的，即为顺序随机。OI模式适用于**成对模式**（使用简单的数据结构，减少Reducer内存使用），因为如果在Reducer中对数据排序是在内存中进行的。

## 举例

一个组合键$(K_1, K_2)$的归约器，其中 $K_1$是这个组合键的自然键部分，Reducer接收到的值为：$(V_1, V_2,...,V_n)$，通过实现OI模式，可以对到达归约器的值进行分类和排序，使用OI模式的唯一目的是确定提供给Reducer的数据顺序。

OI设计模式Reducer输入数据转如下，保证键$[(K_1, K_{2a}),(K_1, K_{2b}),(K_1, K_{2c})]$进入同一个Reducer：

| 组合键          | 值                        |
| --------------- | ------------------------- |
| $(K_1, K_{2a})$ | $\{A_1, A_2, \dots,A_m\}$ |
| $(K_1, K_{2b})$ | $\{B_1, B_2, \dots,B_p\}$ |
| $(K_1, K_{2c})$ | $\{C_1, C_2, \dots,C_q\}$ |

在上面表中

- $m+p+q=n$；
- 排序顺序：$K_{2a} > K_{2b} > K_{2c}$或者$K_{2a} < K_{2b} < K_{2c}$，对应生序与降序；
- $A_i,B_j,C_k \in \{V_1, V_2, \dots, V_n\}$

适当的利用OI模式可以实现对Reducer数据输入的值的排序：
$$
\{A_1, A_2, \dots,A_m, B_1, B_2, \dots,B_p,C_1, C_2, \dots,C_q\}
$$
 如何将组合键$(K_1, K_2)$根据自然键将数据映射到同一个分区？**自定义分区器**，只关注组合键中自然键的部分。

## 语料集中相对词频统计

使用相对词频统计来对OI模式的练习。我们的目标是建立一个$N \times N$的矩阵$M$，$N$为语料集分词后的词向量，单元$M_{ij}$表示词$(N_i,N_j)$在上下文窗口中出现的次数。形式化描述如下：

定义定义语料集词向量为：
$$
W_1, W_2, W_3, \dots, W_6
$$
上下文窗口长度为$2$，单词$W_i$的临域表为：

| 单词  | 临域$\pm$2        |
| ----- | ----------------- |
| $W_1$ | $W_2,W_3$         |
| $W_2$ | $W_1,W_3,W_4$     |
| $W_3$ | $W_1,W_2,W_4,W_5$ |
| $W_4$ | $W_2,W_3,W_5,W_6$ |
| $W_5$ | $W_3,W_4,W_6$     |
| $W_6$ | $W_4,W_5$         |

计算单词相对频度需要边缘计数，即矩阵的行和列总和，计算公式如下：
$$
f(W_j|W_i) = \frac{N(W_i,W_j)}{\sum_{k=1}^{n}N(W_i,W_k)}
$$
在MapReduce和Spark中如何求解呢？具体实现方案见后续章节。

### Hadoop/MapReduce实现反转排序

语料集相对词频求解需要：1. 单词$W_i$在上下文窗口中出现的总次数，使用组合键$(W_i, *)$表示；2. 单词$W_i$与某个单词$W_j$在上线文窗口出现的总次数，使用组合键$(W_i,W_j)$表示。

Reducer应该接受到下表有序数据：

| 键           | 值(出现次数)         |
| ------------ | -------------------- |
| $(W_i,*)$    | $A_1,A_2,\dots, A_m$ |
| $(W_i, W_1)$ | $B_1,B_2, \dots,B_p$ |
| $(W_i, W_2)$ | $C_1,C_2, \dots,C_q$ |
| $\dots$      | $ \dots $            |

Hadoop/MapReduce实现需要自定义组合键类，定制分区器类，combiner，代码具体实现类如下：

| 类名                      | 类描述                                                       |
| ------------------------- | ------------------------------------------------------------ |
| PairOfWords               | 组合键$(K_1,K_2)$，注意MapReduce的键需要实现comparable，用于进入Reducer的排序 |
| RelativeFrequencyMapper   | 映射器，将输入数据转换为键值对形式                           |
| RelativeFrequencyReducer  | 归约器，对同一个分区的数据进行计算                           |
| OrderInversionPartitioner | 定制分区器，将Mapper的输出数据映射到Reducer，分区数对应Reducer数目，多个Reducer可以并行处理，需要注意Reducer OOM问题 |
| OrderInversionCombiner    | combine                                                      |
| RelativeFrequencyDriver   | 提交作业驱动器                                               |

### MapReduce combiner

Hadoop框架使用Mapper将数据处理成一个<key,value>键值对，再网络节点间对其进行整理(shuffle)，然后使用Reducer处理数据并进行最终输出。  在上述过程中，我们看到至少两个性能瓶颈：

1. 如果我们有10亿个数据，Mapper会生成10亿个键值对在网络间进行传输，但如果我们只是对数据求最大值，那么很明显的Mapper只需要输出它所知道的最大值即可。这样做不仅可以减轻网络压力，同样也可以大幅度提高程序效率。
2. 如果数据源远远不是一致性的或者说平衡分布的，这样不仅Mapper中的键值对、中间阶段(shuffle)的键值对等，大多数的键值对最终会聚集于一个单一的Reducer之上，压倒这个Reducer，从而大大降低程序的性能。

Mapreduce中的Combiner就是为了避免map任务和reduce任务之间的数据传输而设置的，Hadoop允许用户针对map task的输出指定一个合并函数。即为了减少传输到Reduce中的数据量。它主要是为了削减Mapper的输出从而减少网络带宽和Reducer之上的负载。

数据格式转换：

```xml
map: (K1, V1) → list(K2,V2) 
combine: (K2, list(V2)) → list(K3, V3) 
reduce: (K3, list(V3)) → list(K4, V4)
```

注意：combine的输入和reduce的完全一致，输出和map的完全一致。

对于Combiner有几点需要说明的是：

1）有很多人认为这个combiner和map输出的数据合并是一个过程，其实不然，map输出的数据合并只会产生在有数据spill出的时候，即进行merge操作。

2）与mapper与reducer不同的是，combiner没有默认的实现，需要显式的设置在conf中才有作用。

3）并不是所有的job都适用combiner，只有操作满足结合律的才可设置combiner。combine操作类似于：opt(opt(1, 2, 3), opt(4, 5, 6))。如果opt为求和、求最大值的话，可以使用，但是如果是求中值的话，不适用。

4）一般来说，combiner和reducer它们俩进行同样的操作。

### Spark实现

