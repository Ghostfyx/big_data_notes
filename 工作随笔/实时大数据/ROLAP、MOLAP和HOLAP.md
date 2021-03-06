# ROLAP、MOLAP和HOLAP

## 1. OLAP简介

OLAP(on-Line Analysis Processing)是使分析人员、管理人员或执行人员能够从多角度对信息进行快速、一致、交互地存取，从而获得对数据的更深入了解的一类软件技术。OLAP的核心概念是“维”（dimension)，维是人们观察客观世界的角度，是一种高层次的类型划分。

 OLAP的基本多维分析操作有钻取(roll up和drill down)、切片(slice)和切块(dice)、以及旋转(pivot)、drill acros、drill through等

  1）钻取

改变维的层次，变换分析的粒度。它包括向上钻取和向下钻取。roll up是在某一维上低层次的细节数据概括到高层次的汇总数据，或者减少维数；而drill down则相反，它从汇总数据深入到细节数据进行观察或增加新维。

2）切片和切块

在一部分维上选定值后，关心度量数据在剩余维上得分布。如果剩余的维只有两个，则是切片；如果有三个，则是切块。

3）**旋转**

旋转是变换维的方向，类似于行列互换。

OLAP有多种实现方法，根据存储数据的方式不同可以分为ROLAP、MOLAP、HOLAP

## 2. ROLAP、MOLAP、HOLAP

| **名称**                     | **描述**                   | **细节数据存储位置** | **聚合后的数据存储位置** |
| ---------------------------- | -------------------------- | -------------------- | ------------------------ |
| ROLAP(Relational OLAP)       | 基于关系数据库的OLAP实现   | 关系型数据库         | 关系型数据库             |
| MOLAP(Multidimensional OLAP) | 基于多维数据组织的OLAP实现 | 数据立方体           | 数据立方体               |
| HOLAP(Hybrid OLAP)           | 基于混合数据组织的OLAP实现 | 关系型数据库         | 数据立方体               |

