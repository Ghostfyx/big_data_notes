# 第六章 处理不同的数据类型

第五章介绍了关于DataFrame的基本概念和抽象。本章将介绍表达式的构建，这是Spark结构化操作的基础，还将介绍对不同数据类型的处理，主要包括以下内容：

- 布尔型
- 数字型
- 字符串型
- 日期和时间戳类型
- 空值处理
- 复杂类型
- 用户自定义函数

## 6.1 处理布尔类型

布尔类型在数据分析中至关重要，因为它是所有过滤操作的基础。布尔语句由四个要素组成：and、or、true或false。基于这些简单要素来构建返回值为true或false的逻辑语句。这些语句经常被作为过滤条件。

基于零售数据集说明使用布尔值的方法。可以指定相等以及小于或大于：

```scala
// in scala

import org.apache.spark.sql.functions.col
df.where(col("InvoiceNo").equalTo(536365))
.select("InvoiceNo", "Description")
.show(5, false)
```

------

Scala对于==和===的使用具有一些特殊的语义。 在Spark中，如果要按相等过滤，则应使用 ===(等于)或 =!= (不等于)。 还可以使用not函数和equal To方法。

------

```scala
// in Scala
import org.apache.spark.sql.functions.col
df.where(col("InvoiceNo") === 536365)
.select("InvoiceNo", "Description")
.show(5, false)
```

```python
# in Python
from pyspark.sql.functions import col
df.where(col("InvoiceNo") != 536365)\
.select("InvoiceNo", "Description")\
.show(5, False)
```

```
+---------+-----------------------------+
|InvoiceNo|       Description           |
+---------+-----------------------------+
|  536366 |    HAND WARMER UNION JACK   |
...
|  536367 |   POPPY'S PLAYHOUSE KITCHEN |
+---------+-----------------------------+
```

另一种方法是使用字符串形式的谓词表达式，这对Python或Scala有效。请注意，这里可以使用另一种表示“不相等”的方式：

```python
df.where("InvoiceNo = 536365").show(5, false)

df.where("InvoiceNo <> 536365").show(5, false)
```

可以使用and或者or将多个Boolean表达式连接起来，但是在Spark中，最好是以链式连接的方式组合起来，形成顺序执行的过滤器。

这样做的原因是因为即使Boolean语句是顺序表达的，Spark也会将所有这些过滤器合并为一条语句，并执行这些过滤器，创建and语句

```scala
// in Scala
val priceFilter = col("UnitPrice") > 600
val descripFilter = col("Description").contains("POSTAGE")
df.where(col("StockCode").isin("DOT")).where(priceFilter.or(descripFilter))
.show()
```

```python
# in Python
from pyspark.sql.functions import instr
priceFilter = col("UnitPrice") > 600
descripFilter = instr(df.Description, "POSTAGE") >= 1
df.where(df.StockCode.isin("DOT")).where(priceFilter | descripFilter).show()
```

```sql
-- in SQL
SELECT * FROM dfTable WHERE StockCode in ("DOT") AND(UnitPrice > 600 OR
instr(Description, "POSTAGE") >= 1)
```

布尔表达式不一定非要在过滤器中使用，想要过滤DataFrame，也可以设定Boolean类型的列：

```scala
// in Scala
val DOTCodeFilter = col("StockCode") === "DOT"
val priceFilter = col("UnitPrice") > 600
val descripFilter = col("Description").contains("POSTAGE")
df.withColumn("isExpensive", DOTCodeFilter.and(priceFilter.or(descripFilter)))
.where("isExpensive")
.select("unitPrice", "isExpensive").show(5)
```

```python
# in Python
from pyspark.sql.functions import instr
DOTCodeFilter = col("StockCode") == "DOT"
priceFilter = col("UnitPrice") > 600
descripFilter = instr(col("Description"), "POSTAGE") >= 1
df.withColumn("isExpensive", DOTCodeFilter & (priceFilter | descripFilter))\
.where("isExpensive")\
.select("unitPrice", "isExpensive").show(5)
```

```sql
-- in SQL
SELECT UnitPrice, (StockCode = 'DOT' AND
(UnitPrice > 600 OR instr(Description, "POSTAGE") >= 1)) as isExpensive
FROM dfTable
WHERE (StockCode = 'DOT' AND
(UnitPrice > 600 OR instr(Description, "POSTAGE") >= 1))
```

注意并没有将过滤器设置为一条语句，使用一个列名无需其他工作就可以实现。

将过滤器表示为SQL语句比使用编程式的DataFrame接口更加简单，同时Spark SQL实现这点并不会造成性能下降，例如，以下两条语句是等价的：

```scala
// in Scala
import org.apache.spark.sql.functions.{expr, not, col}

df.withColumn("isExpensive", not(col("UnitPrice").leq(250)))
.filter("isExpensive")
.select("Description", "UnitPrice").show(5)

df.withColumn("isExpensive", expr("NOT UnitPrice <= 250"))
.filter("isExpensive")
.select("Description", "UnitPrice").show(5)
```

------

创建布尔表达式时，如果要处理空值数据则会出现问题。如果数据存在空值，则需要以不同的方式处理了，下面这条语句可以保证执行空值安全的等价测试：

```python
df.where(col("Description").eqNullSafe("hello")).show()
```

## 6.2 处理数值类型

在处理大数据时，过滤之后要执行的第二个常见的任务是计数，在大多数情况下，只需要简单地表达计算方法，并且确保计算方法对于数值类型数据是正确可行的。

举例如下：假设发现错误记录了零售数据中的数量，而真实数量其实等于(当前数量*单位价格)^2 +5，这需要调用一个数值计算函数——pow函数，来对指定列进行幂运算：

```scala
// in Scala
import org.apache.spark.sql.functions.{expr, pow}
val fabricatedQuantity = pow(col("Quantity") * col("UnitPrice"), 2) + 5
df.select(expr("CustomerId"), fabricatedQuantity.alias("realQuantity")).show(2)
```

```python
# in Python
from pyspark.sql.functions import expr, pow
fabricatedQuantity = pow(col("Quantity") * col("UnitPrice"), 2) + 5
df.select(expr("CustomerId"), fabricatedQuantity.alias("realQuantity")).show(2)
```

```
+----------+------------------+
|CustomerId|   realQuantity   |
+----------+------------------+
| 17850.0  |239.08999999999997|
| 17850.0  |     418.7156     |
+----------+------------------+
```

注意：可以对两列数值类型数据进行乘法、加法与减法操作，也可以使用SQL表达式来实现所有这些操作：

```scala
// in Scala
df.selectExpr(
"CustomerId",
"(POWER((Quantity * UnitPrice), 2.0) + 5) as realQuantity").show(2)
```

```python
# in Python
df.selectExpr(
"CustomerId",
"(POWER((Quantity * UnitPrice), 2.0) + 5) as realQuantity").show(2)
```

```sql
-- in SQL
SELECT customerId, (POWER((Quantity * UnitPrice), 2.0) + 5) as realQuantity
FROM dfTable
```

使用round函数与bound函数对小数进行向上取整与向下取整：

```scala
// in Scala
import org.apache.spark.sql.functions.{round, bround}
df.select(round(col("UnitPrice"), 1).alias("rounded"), col("UnitPrice")).show(5)
```

```python
# in python
from pyspark.sql.functions import lit, round, bround
df.select(round(lit("2.5")), bround(lit("2.5"))).show(2)
```

使用函数以及DataFrame统计方法实现计算两列相关性操作：

```scala
// in Scala
import org.apache.spark.sql.functions.{corr}
df.stat.corr("Quantity", "UnitPrice")
df.select(corr("Quantity", "UnitPrice")).show()
```

```python
# in Python
from pyspark.sql.functions import corr
df.stat.corr("Quantity", "UnitPrice")
df.select(corr("Quantity", "UnitPrice")).show()
```

```sql
-- in SQL
SELECT corr(Quantity, UnitPrice) FROM dfTable
```

```
+-------------------------+
|corr(Quantity, UnitPrice)|
+-------------------------+
|   -0.04112314436835551  |
+-------------------------+
```

使用describe函数计算所有数值类型的计数、标准差、最大值和最小值：

```scala
// in Scala
df.describe().show()
```

```python
# in Python
df.describe().show()
```

```
+-------+------------------+------------------+------------------+
|summary|    Quantity      | UnitPrice        |   CustomerID     |
+-------+------------------+------------------+------------------+
| count |       3108       |   3108           |         1968     |
| mean  | 8.627413127413128|4.151946589446603 |15661.388719512195|
| stddev|26.371821677029203|15.638659854603892|1854.4496996893627|
| min   |        -24       |     0.0          |     12431.0      |
| max   |        600       |     607.49       |     18229.0      |
+-------+------------------+------------------+------------------+
```

StatFunction包中封装了许多可供使用的统计函数。

## 6.3 处理字符串类型

字符串操作几乎在每个数据流中都有，可能正在执行正则表达式提取或替换日志文件操作，或者检查其中是否包含简单的字符串，或者将所有字符串都变成大写或者小写。

当一个给定的字符串中每个单词之间用空格隔开时，`initcap`函数会将每个单词的首字母大写。

```
// in Scala
import org.apache.spark.sql.functions.{initcap}
df.select(initcap(col("Description"))).show(2, false)

# in Python
from pyspark.sql.functions import initcap
df.select(initcap(col("Description"))).show()

-- in SQL
SELECT initcap(Description) FROM dfTable
```

```
+----------------------------------+
|       initcap(Description)       |
+----------------------------------+
|White Hanging Heart T-light Holder|
|        White Metal Lantern       |
+----------------------------------+
```

大小写转换：

```
// in Scala
import org.apache.spark.sql.functions.{lower, upper}
df.select(col("Description"),
lower(col("Description")),
upper(lower(col("Description")))).show(2)

# in Python
from pyspark.sql.functions import lower, upper
df.select(col("Description"),
lower(col("Description")),
upper(lower(col("Description")))).show(2)

-- in SQL
SELECT Description, lower(Description), Upper(lower(Description)) FROM dfTable
```

```
+--------------------+--------------------+-------------------------+
|    Description     | lower(Description) |upper(lower(Description))|
+--------------------+--------------------+-------------------------+
|WHITE HANGING HEA...|white hanging hea...|   WHITE HANGING HEA...  |
| WHITE METAL LANTERN| white metal lantern|   WHITE METAL LANTERN   |
+--------------------+--------------------+-------------------------+
```

删除字符串周围的空格或在其周围添加空格，可以使用`lpad`，`ltrim`，`rpad` 和 `rtrim`，`trim`

```scala
// in Scala
import org.apache.spark.sql.functions.{lit, ltrim, rtrim, rpad, lpad, trim}
df.select(
ltrim(lit(" HELLO ")).as("ltrim"),
rtrim(lit(" HELLO ")).as("rtrim"),
trim(lit(" HELLO ")).as("trim"),
lpad(lit("HELLO"), 3, " ").as("lp"),
rpad(lit("HELLO"), 10, " ").as("rp")).show(2)
```

```python
# in Python
from pyspark.sql.functions import lit, ltrim, rtrim, rpad, lpad, trim
df.select(
ltrim(lit(" HELLO ")).alias("ltrim"),
rtrim(lit(" HELLO ")).alias("rtrim"),
trim(lit(" HELLO ")).alias("trim"),
lpad(lit("HELLO"), 3, " ").alias("lp"),
rpad(lit("HELLO"), 10, " ").alias("rp")).show(2)
```

```sql
-- in SQL
SELECT
ltrim(' HELLLOOOO '),
rtrim(' HELLLOOOO '),
trim(' HELLLOOOO '),
lpad('HELLOOOO ', 3, ' '),
rpad('HELLOOOO ', 10, ' ')
FROM dfTable
```

```
+---------+---------+-----+---+----------+
|   ltrim |  rtrim  | trim| lp|    rp    |
+---------+---------+-----+---+----------+
|  HELLO  |  HELLO  |HELLO| HE|HELLO     |
|  HELLO  |  HELLO  |HELLO| HE|HELLO     |
+---------+---------+-----+---+----------+
```

## 6.3 正则表达式

最常见的任务之一是在一个字传中搜索子串、替换被选中的字符串等。使用正则表达式实现这些功能，正则表达式使得使用可以指定一组规则，从字符串中提取子串或替换子串。

为了执行正则表达式任务，Spark中需要两个关键功能：regexp_extract和regexp_replace。这些函数分别提取值和替换值。

```scala
// in Scala
import org.apache.spark.sql.functions.regexp_replace
val simpleColors = Seq("black", "white", "red", "green", "blue")
val regexString = simpleColors.map(_.toUpperCase).mkString("|")
// the | signifies `OR` in regular expression syntax
df.select(
regexp_replace(col("Description"), regexString, "COLOR").alias("color_clean"),
col("Description")).show(2)
```

```python
# in Python
from pyspark.sql.functions import regexp_replace
regex_string = "BLACK|WHITE|RED|GREEN|BLUE"
df.select(
regexp_replace(col("Description"), regex_string, "COLOR").alias("color_clean"),
col("Description")).show(2)
```

```sql
-- in SQL
SELECT
regexp_replace(Description, 'BLACK|WHITE|RED|GREEN|BLUE', 'COLOR') as
color_clean, Description
FROM dfTable
```

```
+--------------------+--------------------+
|    color_clean     |     Description    |
+--------------------+--------------------+
|COLOR HANGING HEA...|WHITE HANGING HEA...|
| COLOR METAL LANTERN| WHITE METAL LANTERN|
+--------------------+--------------------+
```

用其他字符替换给定的字符。 将其构建为正则表达式可能很繁琐，因此Spark还提供了 `translation` 函数来替换这些值。 这是在字符级别完成的，它将用替换字符串中的索引字符替换字符的所有实例：

```scala
// in Scalaimport 
org.apache.spark.sql.functions.translate
df.select(translate(col("Description"), "LEET", "1337"), col("Description"))
.show(2)
```

```python
# in Python
from pyspark.sql.functions import translate
df.select(translate(col("Description"), "LEET", "1337"),col("Description"))\
.show(2)
```

```
-- in SQL
SELECT translate(Description, 'LEET', '1337'), Description FROM dfTable
```

```
+----------------------------------+--------------------+
|translate(Description, LEET, 1337)|    Description     |
+----------------------------------+--------------------+
|   WHI73 HANGING H3A...           |WHITE HANGING HEA...|
|   WHI73 M37A1 1AN73RN            | WHITE METAL LANTERN|
+----------------------------------+--------------------+
```

拉出第一个提到的颜色：

```
// in Scala
import org.apache.spark.sql.functions.regexp_extract
val regexString = simpleColors.map(_.toUpperCase).mkString("(", "|", ")")
// the | signifies OR in regular expression syntax
df.select(
regexp_extract(col("Description"), regexString, 1).alias("color_clean"),
col("Description")).show(2)

# in Python
from pyspark.sql.functions import regexp_extract
extract_str = "(BLACK|WHITE|RED|GREEN|BLUE)"
df.select(
regexp_extract(col("Description"), extract_str, 1).alias("color_clean"),
col("Description")).show(2)

-- in SQL
SELECT regexp_extract(Description, '(BLACK|WHITE|RED|GREEN|BLUE)', 1),
Description FROM dfTable
+-------------+--------------------+
| color_clean |     Description    |
+-------------+--------------------+
|    WHITE    |WHITE HANGING HEA...|
|    WHITE    | WHITE METAL LANTERN|
+-------------+--------------------+
```

contains方法检查每列是否包含指定字符串，该方法返回一个布尔值：

```scala
// in Scala
val containsBlack = col("Description").contains("BLACK")
val containsWhite = col("DESCRIPTION").contains("WHITE")
df.withColumn("hasSimpleColor", containsBlack.or(containsWhite))
.where("hasSimpleColor")
.select("Description").show(3, false)
```

在Python和SQL中，可以使用 `instr` 函数：

```python
# in Python
from pyspark.sql.functions import instr
containsBlack = instr(col("Description"), "BLACK") >= 1
containsWhite = instr(col("Description"), "WHITE") >= 1
df.withColumn("hasSimpleColor", containsBlack | containsWhite)\
.where("hasSimpleColor")\
.select("Description").show(3, False)
```

```sql
-- in SQL
SELECT Description FROM dfTable
WHERE instr(Description, 'BLACK') >= 1 OR instr(Description, 'WHITE') >= 1
```

```
+----------------------------------+
|            Description           |
+----------------------------------+
|WHITE HANGING HEART T-LIGHT HOLDER|
|WHITE METAL LANTERN               |
|RED WOOLLY HOTTIE WHITE HEART.    |
+----------------------------------+
```

