# 第十一章 基于MLlib的机器学习

MLlib是Spark中提供机器学习函数的库，是专为集群上并行运行的情况而设计的。MLlib包含很多机器学习算法，可以在Spark支持的所有编程语言中使用。本章会展示如何在程序中调用 MLlib，并且给出一些常用的使用技巧。

## 1.1 概述

MLlib的设计理念非常简单：把数据以RDD的形式表示，然后在分布书数据集上调用各种算法。MLlib引入了一些数据类型(比如点和向量)，不过归根结底，MLlib 就是 RDD 上一系列可供调用的函数的集合。比如，如果要用 MLlib 来完成文本分类的任务(例如识 别垃圾邮件)，只需要按如下步骤操作：

（1）读取文本数据集，使用字符串RDD来表示。

（2）运行MLlib中的一个特征提取(feature extraction)算法来把文本数据转换为数值特征，结果返回一个向量RDD。

（3）对向量 RDD 调用分类算法(比如逻辑回归)；返回一个模型对象，可以使用该对象对新的数据点进行分类。

（4）使用MLlib的评估函数在测试数据集上评估模型。

**注意：**MLlib中只包含能够在集群上运行良好的并行算法。有些机器学习的经典学习算法没有包含其中，就是因为它们不支持并行执行。一些较新的研究得出的算法因为适用于集群，也被包含在 MLlib 中，例如分布式随机森林算法 (distributed random forests)、K-means|| 聚类、交替最小二乘算法(alternating least squares) 等。这样的选择使得 MLlib 中的每一个算法都适用于大规模数据集。如果要在许多小规模数据集上训练机器学习模型，最好在各个节点上使用单节点机器学习算法库，例如：Weka，SciKit-Learn，有以下几种具体的使用方式：

- 使用Spark的map( )函数在各节点上使用单节点的机器学习算法库。
- 使用机器学习流水线，对同一算法使用不同参数对小规模数据集分别训练，选出最优模型参数，在 Spark 中，可以通过把参数列表传给 parallelize() 来在不同的节点上分别运行不同的参数，而在每个节点上则使用单节点的机器学习库来实现。

在 Spark 1.2 中，MLlib 引入了流水 线 API，可以用来构建这样的流水线。这套 API和类似SciKit-Learn这样的高层库很像，可以让人很容易地写出完整的、可自动调节的机器学习流水线。本章还是以低级 API 为主，会在本章最后简要介绍一下这套 API。

## 11.2 系统要求

MLlib 需要你的机器预装一些线性代数的库。首先，需要为操作系统安装 gfortran 运行库。如果 MLlib 警告找不到 gfortran 库的话，可以按 MLlib 网站(http://spark.apache.org/docs/latest/mllib-guide.html)上说明的步骤处理。其次，如果要在 Python 中使用MLlib，需要安装 NumPy(http://www.numpy.org/)。

## 11.3 机器学习基础

机器学习算法尝试根据训练数据(training data)使得表示算法行为的数学目标最大化，并 以此来进行预测或作出决定。机器学习问题分为几种，包括分类、回归、聚类，每种都有 不一样的目标。拿分类(classification)作为一个简单的例子:分类是基于已经被标记的其他数据点(比如一些已经分别被标记为垃圾邮件或非垃圾邮件的邮件)作为例子来识别一 个数据点属于几个类别中的哪一种(比如判断一封邮件是不是垃圾邮件)。

所有的学习算法都需要定义每个数据点的特征(feature)集，即传递给学习函数的值。例如，对于一封邮件来说，一些特征可能包括其来源服务器、提到 free 这个单词的次 数、字体颜色等。在很多情况下，正确地定义特征才是机器学习中最有挑战性的部分。例如，在产品推荐的任务中，仅仅加上一个额外的特征(例如意识到推荐给用户的书籍可能也取决于用户看过的电影)，就有可能极大地改进结果。

大多数算法都只是专为数据特征(具体来说，就是一个代表各个特征值的数字向量)定义的，因此特征提取并转换为特征向量是机器学习过程中很重要的一步。例如，在文本分类中(比如垃圾邮件和非垃圾邮件的例子)，有好几个提取文本特征的方法，比如对各个单词出现的频率进行计数。

当数据表示为特征向量的形式之后，大多数机器学习算法都会根据这些向量优化一个定义好的数学函数。例如：某个分类算法可能会在特征向量的空间中定义一个平面(SVM算法)，使得这个平面能“最好”地分隔垃圾邮件和非垃圾邮件。这里需要为“最好”给出定义(比如大 多数数据点都被这个平面正确分类)。算法会在运行结束时返回一个代表学习决定的模型，这个模型就可以用来对新的点进行预测，图 11-1 展示了一个机器学习流水线的 示例。

![](./img/11-1.jpg)

​												**图 11-1:机器学习流水线中的典型步骤**

最后，大多数机器学习算法都会有多个影响结果的参数，所以现实中的机器学习流水线会被训练出多个不同版本的模型，然后分别对其进行评估。通常需要把 输入数据分为“训练集”和“测试集”，并且只使用前者进行训练，这样就可以用后者来检 验模型是否过度拟合(overfit)了训练数据。MLlib 提供了几个算法来进行模型评估。

**示例：垃圾邮件分类**

举一个用来快速了解 MLlib 的例子，我们会展示一个构建垃圾邮件分类器的简单 程序(见例 11-1 至例 11-3)。这个程序使用了 MLlib 中的两个函数：HashingTF与LogisticRegresstionWithSGD，前者从文本数据构建词频(term frequency)特征向量，后者使用随机梯度下降法(Stochastic Gradient Descent，简称 SGD)实现逻辑回归。

假设从两个文件 spam.txt 与 normal.txt 开始，两个文件分别包含垃圾邮件和非垃圾邮件的例子， 每行一个。接下来我们就根据词频把每个文件中的文本转化为特征向量，然后训练出一个可以把两类消息分开的逻辑回归模型。

**例 11-1:Python 版垃圾邮件分类器**

```python
from pyspark.mllib.regression import LabeledPoint
from pyspark.mllib.feature import HashingTF
from pyspark.mllib.classification import LogisticRegressionWithSGD

spam = sc.textFile("spam.txt")
normal = sc.textFile("normal.txt")
# 创建一个HashingTF实例来把邮件文本映射为包含10000个特征的向量
tf = HashingTF(numFeatures = 10000)
# 各邮件都被切分为单词，每个单词被映射为一个特征
spamFeatures = spam.map(lambda email: tf.transform(email.split(" "))) normalFeatures = normal.map(lambda email: tf.transform(email.split(" ")))
# 创建LabeledPoint数据集分别存放阳性(垃圾邮件)和阴性(正常邮件)的例子 
positiveExamples = spamFeatures.map(lambda features: LabeledPoint(1, features)) negativeExamples = normalFeatures.map(lambda features: LabeledPoint(0, features))
trainingData = positiveExamples.union(negativeExamples)
trainingData.cache() # 因为逻辑回归是迭代算法，所以缓存训练数据RDD
# 使用SGD算法运行逻辑回归
model = LogisticRegressionWithSGD.train(trainingData)
# 以阳性(垃圾邮件)和阴性(正常邮件)的例子分别进行测试。首先使用
# 一样的HashingTF特征来得到特征向量，然后对该向量应用得到的模型
posTest = tf.transform("O M G GET cheap stuff by sending money to ...".split(" ")) negTest = tf.transform("Hi Dad, I started studying Spark the other ...".split(" ")) print "Prediction for positive test example: %g" % model.predict(posTest)
print ("Prediction for negative test example: %g" % model.predict(negTest))
```

**例 11-2:Scala 版垃圾邮件分类器**

```scala
import org.apache.spark.mllib.regression.LabeledPoint
import org.apache.spark.mllib.feature.HashingTF
import org.apache.spark.mllib.classification.LogisticRegressionWithSGD
    val spam = sc.textFile("spam.txt")
    val normal = sc.textFile("normal.txt")
// 创建一个HashingTF实例来把邮件文本映射为包含10000个特征的向量
val tf = new HashingTF(numFeatures = 10000)
// 各邮件都被切分为单词，每个单词被映射为一个特征
val spamFeatures = spam.map(email => tf.transform(email.split(" ")))
val normalFeatures = normal.map(email => tf.transform(email.split(" ")))
// 创建LabeledPoint数据集分别存放阳性(垃圾邮件)和阴性(正常邮件)的例子
val positiveExamples = spamFeatures.map(features => LabeledPoint(1, features)) 
val negativeExamples = normalFeatures.map(features => LabeledPoint(0, features)) 
val trainingData = positiveExamples.union(negativeExamples)
trainingData.cache() // 因为逻辑回归是迭代算法，所以缓存训练数据RDD
// 使用SGD算法运行逻辑回归
val model = new LogisticRegressionWithSGD().run(trainingData)
// 以阳性(垃圾邮件)和阴性(正常邮件)的例子分别进行测试 
val posTest = tf.transform("O M G GET cheap stuff by sending money to ...".split(" "))
val negTest = tf.transform("Hi Dad, I started studying Spark the other ...".split(" "))
println("Prediction for positive test example: " + model.predict(posTest))
println("Prediction for negative test example: " + model.predict(negTest))
```

**例 11-3:Java 版垃圾邮件分类器**

```java
import org.apache.spark.mllib.classification.LogisticRegressionModel;
import org.apache.spark.mllib.classification.LogisticRegressionWithSGD;
import org.apache.spark.mllib.feature.HashingTF;
import org.apache.spark.mllib.linalg.Vector;
import org.apache.spark.mllib.regression.LabeledPoint;


JavaRDD<String> spam = sc.textFile("spam.txt");	
JavaRDD<String> normal = sc.textFile("normal.txt");
// 创建一个HashingTF实例来把邮件文本映射为包含10000个特征的向量 
final HashingTF tf = new HashingTF(10000);
JavaRDD<LabeledPoint> posExamples = spam.map(new Function<String, LabeledPoint>() {
      public LabeledPoint call(String email) {
        return new LabeledPoint(1, tf.transform(Arrays.asList(email.split(" "))));
} });
JavaRDD<LabeledPoint> negExamples = normal.map(new Function<String, LabeledPoint>() { public LabeledPoint call(String email) {
        return new LabeledPoint(0, tf.transform(Arrays.asList(email.split(" "))));
      }
});
JavaRDD<LabeledPoint> trainData = positiveExamples.union(negativeExamples); trainData.cache(); // 因为逻辑回归是迭代算法，所以缓存训练数据RDD
// 使用SGD算法运行逻辑回归
LogisticRegressionModel model = new LogisticRegressionWithSGD().run(trainData.rdd());
// 以阳性(垃圾邮件)和阴性(正常邮件)的例子分别进行测试 
Vector posTest = tf.transform(Arrays.asList("O M G GET cheap stuff by sending money to ...".split(" ")));
Vector negTest = tf.transform(Arrays.asList("Hi Dad, I started studying Spark the other ...".split(" ")));
System.out.println("Prediction for positive example: " + model.predict(posTest));
System.out.println("Prediction for negative example: " + model.predict(negTest));
```

**PS：**LabeledPoints是MLlib 中用来存放特征向量和对应标签 的数据类型。

## 11.4 数据类型

MLlib包含一些特有的数据类型，它们位于 org.apache.spark.mllib 包(Java/Scala)或pyspark.mllib(Python)内。主要的几个如下：

- vector

一个数学向量，MLlib 既支持稠密向量也支持稀疏向量，前者表示向量的每一位都存储 下来，后者则只存储非零位以节约空间。向量可以通过 mllib.linalg.Vectors 类创建出来。

- LabeledPoint

在诸如分类和回归这样的监督式学习(supervised learning)算法中，LabeledPoint用来表示带标签的数据点，它包含一个特征向量和一个标签，位于mllib.regression 包中。

- Rating

用户对于一个产品的评分，在mllib.recommendation包中，用于产品推荐。

- 各种Model类

每个model都是训练算法的结果，一般有一个predict( )方法可以用来对新的数据点或者数据点组成的RDD应用该模型进行预测。

大多数算法直接操作由 Vector、LabeledPoint 或 Rating 对象组成的 RDD。可以用任意方式创建出这些对象，不过一般来说需要通过对外部数据进行转化操作来构建出RDD——例如，通过读取一个文本文件或者运行一条Spark SQL命令。接下来，使用 map() 将你的数据对象转为 MLlib 的数据类型。

**操作向量**

作为 MLlib 中最常用的数据类型，Vector 类有一些需要注意的地方。

- 向量有两种：稠密向量与稀疏向量。稠密向量把所有维度的值存放在一个浮点数组中，例如，一个 100 维的向量会存储 100 个双精度浮点数。相比之下，稀疏向量只把各 维度中的非零值存储下来。当最多只有**10%**的元素为非零元素时，通常更倾向于使用稀疏向量(不仅是出于对内存使用的考虑，也是出于对速度的考虑)。许多特征提取技术都会生成非常稀疏的向量，所以这种方式常常是一种很关键的优化手段。
- 创建向量的方式在各个语言中有一些细微的差别。在 Python 中，传递的NumPy数组都表示一个稠密向量，也可以使用 mllib.linalg.Vectors 类创建其他类型的向量(如例 11-4 所示)。而在 Java 和 Scala 中，都需要使用 mllib.linalg. Vectors 类(如例 11-5 和例 11-6 所示)。

**例 11-4:用 Python 创建向量**

```python
from numpy import array
from pyspark.mllib.linalg import Vectors

# 创建稠密向量<1.0, 2.0, 3.0>
denseVec1 = array([1.0, 2.0, 3.0]) # NumPy数组可以直接传给MLlib 
denseVec2 = Vectors.dense([1.0, 2.0, 3.0]) # 或者使用Vectors类来创建

# 创建稀疏向量<1.0, 0.0, 2.0, 0.0>;该方法只接收
# 向量的维度(4)以及非零位的位置和对应的值
# 这些数据可以用一个dictionary来传递，或使用两个分别代表位置和值的list
sparseVec1 = Vectors.sparse(4, {0: 1.0, 2: 2.0})
sparseVec2 = Vectors.sparse(4, [0, 2], [1.0, 2.0])
```

**例 11-5:用 Scala 创建向量**

```scala
import org.apache.spark.mllib.linalg.Vectors
// 创建稠密向量<1.0, 2.0, 3.0>;Vectors.dense接收一串值或一个数组 
val denseVec1 = Vectors.dense(1.0, 2.0, 3.0)
val denseVec2 = Vectors.dense(Array(1.0, 2.0, 3.0))
// 创建稀疏向量<1.0, 0.0, 2.0, 0.0>;该方法只接收
// 向量的维度(这里是4)以及非零位的位置和对应的值
val sparseVec1 = Vectors.sparse(4, Array(0, 2), Array(1.0, 2.0))
```

**例 11-6:用 Java 创建向量**

```java
import org.apache.spark.mllib.linalg.Vector;
import org.apache.spark.mllib.linalg.Vectors;
// 创建稠密向量<1.0, 2.0, 3.0>;Vectors.dense接收一串值或一个数组 
Vector denseVec1 = Vectors.dense(1.0, 2.0, 3.0);
Vector denseVec2 = Vectors.dense(new double[] {1.0, 2.0, 3.0});
// 创建稀疏向量<1.0, 0.0, 2.0, 0.0>;该方法只接收
// 向量的维度(这里是4)以及非零位的位置和对应的值
Vector sparseVec1 = Vectors.sparse(4, new int[] {0, 2}, new double[]{1.0, 2.0});
```

在 Java 和 Scala 中，MLlib 的 Vector 类只是用来为数据表示服务的，而没有在用 户 API 中提供加法和减法这样的向量的算术操作。在 Python 中，你可以对稠密向量使用 NumPy 来进行这些数学操作，也可以把这些操作传给 MLlib。Java和Scala中，可以使用一些第三方的库，比如 Scala 中的 Breez或者 Java 中的 MTJ，然后再把数据转为 MLlib 向量。

## 11.5 算法

本节会介绍 MLlib 中主要的算法，以及它们的输入和输出类型，专注于如何调用并配置这些算法。

### 11.5.1 特征提取

mlib.feature包中包含一些用来进行常见特征转化的类，这些类有从文本(或其他表示)创建特征向量的算法，也有对特征向量进行正规化和伸缩变换的方法。

**TF-IDF**

词频—逆文档频率(简称 TF-IDF)是一种用来从文本文档(例如网页)中生成特征向量的 简单方法。它为文档中的每个词计算两个统计值:一个是词频(TF)，也就是每个词在文 档中出现的次数，另一个是逆文档频率(IDF)，用来衡量一个词在整个文档语料库中出现 的(逆)频繁程度。这些值的积，也就是 TF × IDF，展示了一个词与特定文档的相关程 度(比如这个词在某文档中很常见，但在整个语料库中却很少见)。

MLlib 有两个算法可以用来计算 TF-IDF:HashingTF 和 IDF：

HashingTF从一个文档中计算给定大小的词频向量，为了将词与向量顺序对应起来，使用了哈希法。在类似英语这样的语言中，有几十万个单词，因此将每个 单词映射到向量中的一个独立的维度上需要付出很大代价。而 HashingTF 使用每个单词对 所需向量的长度 *S* 取模得出的哈希值，把所有单词映射到一个 0 到 *S*-1 之间的数字上。由 此我们可以保证生成一个 *S* 维的向量。

**例 11-7 在 Python 中使用了 HashingTF**

```python
from pyspark.mllib.feature import HashingTF
sentence = "hello hello world"
words = sentence.split() # 将句子切分为一串单词
tf = HashingTF(10000) # 创建一个向量，其尺寸S = 10,000 
tf.transform(words)
rdd = sc.wholeTextFiles("data").map(lambda (name, text): text.split()) 
tfVectors = tf.transform(rdd) # 对整个RDD进行转化操作
```

**例 11-8:在 Python 中使用 TF-IDF**

```python
from pyspark.mllib.feature import HashingTF, IDF
# 将若干文本文件读取为TF向量
rdd = sc.wholeTextFiles("data").map(lambda (name, text): text.split()) 
tf = HashingTF()
# RDD缓存
tfVectors = tf.transform(rdd).cache()
# 计算IDF，然后计算TF-IDF向量
idf = IDF()
idfModel = idf.fit(tfVectors)
tfIdfVectors = idfModel.transform(tfVectors)
```

**1. 缩放**

大多数机器学习算法都要考虑特征向量中各元素的幅值，并且在特征缩放调整为平等对待 时表现得最好(例如所有的特征平均值为 0，标准差为 1)。

**例 11-9:在 Python 中缩放向量**

```python
from pyspark.mllib.feature import StandardScaler

vectors = [Vectors.dense([-2.0, 5.0, 1.0]), Vectors.dense([2.0, 0.0, 1.0])]
dataset = sc.parallelize(vectors)
scaler = StandardScaler(withMean=True, withStd=True)
model = scaler.fit(dataset)
result = model.transform(dataset)
# 结果:{[-0.7071, 0.7071, 0.0], [0.7071, -0.7071, 0.0]}
```

**2. 正规化**

在一些情况下，在准备输入数据时，把向量正规化为长度 1 也是有用的。使用 Normalizer 类可以实现，只要使用 Normalizer.transform(rdd) 就可以了。默认情况下，Normalizer 使 用 $L^2$ 范式(也就是欧几里得距离)，不过你可以给 Normalizer 传递一个参数 *p* 来使用$L^p$范式。

**3. word2Vec**

Word2Vec是一个基于神经网络的文本特征化算法， 可以用来将数据传给许多下游算法。Spark 在 mllib.feature.Word2Vec 类中引入了该算法的一个实现。

要训练 Word2Vec，需要传给它一个用 String 类的 Iterable 表示 的语料库。和前面的“TF-IDF”一节所讲的很像，Word2Vec 也推荐对单词进行正规化处 理(例如全部转为小写、去除标点和数字)。

### 11.5.2  统计

不论是在即时的探索中，还是在机器学习的数据理解中，基本的统计都是数据分析的重要 部分。MLlib 通过 mllib.stat.Statistics 类中的方法提供了几种广泛使用的统计函数，这 些函数可以直接在 RDD 上使用。一些常用的函数如下所列。

- Statistics.colStats(RDD)：计算由向量组成的 RDD 的统计性综述，保存着向量集合中每列的最小值、最大值、平均值和方差。
- Statistics.corr(rdd, model)：计算由向量组成的 RDD 中的列间的相关矩阵，使用皮尔森相关(Pearson correlation) 或斯皮尔曼相关(Spearman correlation)中的一种(*method* 必须是 pearson 或 spearman 中的一个)。
- Statistic.chiSqTes(rdd)：计算由 LabeledPoint 对象组成的 RDD 中每个特征与标签的皮尔森独立性测试(Pearson’s independence test) 结 果。 返 回 一 个 ChiSqTestResult 对 象， 其 中 有 p 值 (p-value)、测试统计及每个特征的自由度。标签和特征值必须是分类的(即离散值)。

除了这些算法以外，数值 RDD 还提供几个基本的统计函数，例如 mean()、stdev() 以及 sum()， 6.6 节中有所提及。除此以外，RDD 还支持 sample() 和 sampleByKey()，使 用它们可以构建出简单而分层的数据样本。

### 11.5.3 分类与回归

分类与回归是监督学习的两种主要方式。分类与回归是监督式学习的两种主要形式。监督式学习指算法尝试使用有标签的训练数据 (也就是已知结果的数据点)根据对象的特征预测结果。分类和回归的区别在于预测的变量的类型：在分类中，预测出的变量是离散的(也就是一个在有限集中的值，叫作类别); 比如，分类可能是将邮件分为垃圾邮件和非垃圾邮件，也有可能是文本所使用的语言。在 回归中，预测出的变量是连续的(例如根据年龄和体重预测一个人的身高)。

分类和回归都会使用 MLlib 中的 LabeledPoint 类。一个LabeledPoint类就是由一个label和一个feature向量组成。

**注意：**对于二元分类，MLlib 预期的标签为 0 或 1。在某些情况下可能会使用 -1 和 1，但这会导致错误的结果。对于多元分类，MLlib 预期标签范围是从 0 到 *C*-1，其中 *C* 表示类别数量。

MLlib 包含多种分类与回归的算法，其中包括简单的线性算法以及决策树和森林算法。

**1. 线性回归**

线性回归是回归中最常用的方法之一，是指用特征的线性组合来预测输出值。MLlib 也支持$L^1$和的$L^2$正则的回归，通常称为 Lasso 和 ridge 回归。

线性回归算法可以使用的类包括 mllib.regression.LinearRegressionWithSGD、LassoWithSGD 以及 RidgeRegressionWithSGD。这遵循了 MLlib 中常用的命名模式，即对于涉及多个算法的 问题，在类名中使用“With”来表明所使用的算法。这里，SGD 代表的是随机梯度下降法。

这些类都有几个可以用来对算法进行调优的参数。

- numIterations 要运行的迭代次数(默认值:100)。
- stepSize 梯度下降的步长(默认值:1.0)。
- intercept 是否给数据加上一个干扰特征或者偏差特征——也就是一个值始终为 1 的特征(默认 值:false)。
- regParam Lasso 和 ridge 的正规化参数(默认值:1.0)。

**例 11-10:Python 中的线性回归**

```python
from pyspark.mllib.regression import LabeledPoint
from pyspark.mllib.regression import LinearRegressionWithSGD
points = # (创建LabeledPoint组成的RDD)
model = LinearRegressionWithSGD.train(points, iterations=200, intercept=True) 
print("weights: %s, intercept: %s" % (model.weights, model.intercept))
```

**例 11-11:Scala 中的线性回归**

```scala
import org.apache.spark.mllib.regression.LabeledPoint
import org.apache.spark.mllib.regression.LinearRegressionWithSGD
val points: RDD[LabeledPoint] = // ...
val lr = new LinearRegressionWithSGD().setNumIterations(200).setIntercept(true)
val model = lr.run(points)
println("weights: %s, intercept: %s".format(model.weights, model.intercept))
```

**例 11-12:Java 中的线性回归**

```java
import org.apache.spark.mllib.regression.LabeledPoint;
import org.apache.spark.mllib.regression.LinearRegressionWithSGD;
import org.apache.spark.mllib.regression.LinearRegressionModel;
JavaRDD<LabeledPoint> points = // ...
LinearRegressionWithSGD lr = new 					LinearRegressionWithSGD().setNumIterations(200).setIntercept(true);
// 需要通过调用 .rdd() 方法把 JavaRDD 转为 Scala 中的 RDD 类
LinearRegressionModel model = lr.run(points.rdd());
System.out.printf("weights: %s, intercept: %s\n", model.weights(), model.intercept());
```

**2. 逻辑回归**

逻辑回归是一种二元分类方法，用来寻找一个分隔阴性和阳性示例的线性分割平面。在 MLlib 中，它接收一组标签为 0 或 1 的 LabeledPoint，返回可以预测新点的分类的 LogisticRegressionModel 对象。有两 种可以用来解决逻辑回归问题的算法:SGD 和 LBFGS5。LBFGS 一般是最好的选择。

**3. 支持向量机**

支持向量机(简称 SVM)算法是另一种使用线性分割平面的二元分类算法，同样只预期 0 或 者 1 的标签。通过 SVMWithSGD 类，我们可以访问这种算法，它的参数与线性回归和逻辑回归 的参数差不多。返回的 SVMModel 与 LogisticRegressionModel 一样使用阈值的方式进行预测。

**4. 朴素贝叶斯**

朴素贝叶斯(Naive Bayes)算法是一种多元分类算法，它使用基于特征的线性函数计算将 一个点分到各类中的得分。这种算法通常用于使用 TF-IDF 特征的文本分类，以及其他一 些应用。MLlib 实现了多项朴素贝叶斯算法，需要非负的频次(比如词频)作为输入特征。

在 MLlib 中，你可以通过 mllib.classification.NaiveBayes 类来使用朴素贝叶斯算法。它支 持一个参数 lambda(Python 中是 lambda_)，用来进行平滑化。你可以对一个由 LabeledPoint 组成的 RDD 调用朴素贝叶斯算法，对于 *C* 个分类，标签值范围在 0 至 *C*-1 之间。

返回的 NaiveBayesModel 可以使用 predict() 预测对某点最合适的分类，也可以访问训练好的模型的两个参数:各特征与各分类的可能性矩阵 theta(对于 *C* 个分类和 *D* 个特 征的情况，矩阵大小为 *C* × *D*)，以及表示先验概率的 *C* 维向量 pi。

**5. 决策树与随机森林**

决策树是一个灵活的模型，可以用来进行分类，也可以用来进行回归。决策树以节点树的 形式表示，每个节点基于数据的特征作出一个二元决定(比如，这个人的年龄是否大于 20 ?)，而树的每个叶节点则包含一种预测结果(例如，这个人是不是会买一个产品?)。 决策树的吸引力在于模型本身容易检查，而且决策树既支持分类的特征，也支持连续的特 征。图 11-2 展示了一个决策树的示例。

![](./img/11-2.jpg)

​												**图 11-2:一个预测用户是否会购买一件产品的决策树示例**

在 MLlib 中，你可以使用 mllib.tree.DecisionTree 类中的静态方法 trainClassifier() 和 trainRegressor() 来训练决策树。和其他有些算法不同的是，Java 和 Scala 的 API 也使用 静态方法，而不使用 setter 方法定制的 DecisionTree 对象。该训练方法接收如下所列参数。

- data 由LabeledPoint组成RDD
- numClasses(仅用于分类时)要使用的类别数量
- maxDepth 树的最大深度(默认值：5)
- maxBins 在构建各节点时将数据分到多少个箱子中(推荐值：32)
- categoricalFeaturesInfo 一个映射表，用来指定哪些特征是分类的，以及它们各有多少个分类。例如，如果特征 1 是一个标签为 0 或 1 的二元特征，特征 2 是一个标签为 0、1 或 2 的三元特征，你就 应该传递 {1: 2, 2: 3}。如果没有特征是分类的，就传递一个空的映射表。

RandomForest 类，可以用 来构建一组树的组合，也被称为随机森林。它可以通过 RandomForest.trainClassifier 和 trainRegressor 使用。除了刚才列出的每棵树对应的参数外，RandomForest 还接收如下参数。

- numTrees  要构建的树的数量。提高 numTrees 可以降低对训练数据过度拟合的可能性。
- featureSubsetStrategy 在每个节点上需要考虑的特征数量，可以是auto、all、sqrt、log2以及onethird，值越大花费的代价越大
- seed 所使用的随机数种子

### 11.5.4 聚类

聚类算法是一种无监督学习任务，用于将对象分到具有高度相似性的聚类中。前面提到的监督式任务中的数据都是带标签的，而聚类可以用于无标签的数据。该算法主要用于数据 探索(查看一个新数据集是什么样子)以及异常检测(识别与任意聚类都相距较远的点)。

**KMeans**

MLlib 包含聚类中流行的K-means算法，以及一个叫作 K-means||的变种，可以为并行环境提供更好的初始化策略。

K-means最重要的参数是生成的聚类中心的目标数量K，事实上，几乎不可能提前知 道聚类的“真实”数量，所以最佳实践是尝试几个不同的 K 值，直到聚类内部平均距离不 再显著下降为止。然而，算法一次只能接收一个 K 值。除了 K 以外，MLlib 中的 K-means 还接收以下几个参数。

- initialzationMode

	用来初始化聚类中心的方法，可以是“k-means||”或者“random”;k-means||(默认 值)一般会带来更好的结果，但是开销也会略高一些。

- maxIterations

	运行时的最大迭代次数（默认值：100）

- runs

	算法并发运行的数目。MLlib 的 K-means 算法支持从多个起点并发执行，然后选择最佳结果。

### 11.5.4 协同过滤与推荐

协同过滤是一种根据用户对各种产品的交互与评分来推荐新产品的推荐系统技术。协同过 滤吸引人的地方就在于它只需要输入一系列用户 / 产品的交互记录:无论是“显式”的交 互(例如在购物网站上进行评分)还是“隐式”的(例如用户访问了一个产品的页面但是 没有对产品评分)交互皆可。仅仅根据这些交互，协同过滤算法就能够知道哪些产品之间 比较相似(因为相同的用户与它们发生了交互)以及哪些用户之间比较相似，然后就可以作出新的推荐。

**交替最小二乘**

MLlib 中包含交替最小二乘(简称 ALS)的一个实现，这是一个协同过滤的常用算法，可 以很好地扩展到集群上。它位于 mllib.recommendation.ALS 类中。

- rank 使用的特征向量的大小;更大的特征向量会产生更好的模型，但是也需要花费更大的计 算代价(默认值:10)。

-  iterations 要执行的迭代次数(默认值:10)。

- lambda 正则化参数(默认值:0.01)。

- alpha用来在隐式 ALS 中计算置信度的常量(默认值:1.0)。
- numUserBlocks，numProductBlocks 切分用户和产品数据的块的数目，用来控制并行度;你可以传递 -1 来让 MLlib 自动决 定(默认行为)。

