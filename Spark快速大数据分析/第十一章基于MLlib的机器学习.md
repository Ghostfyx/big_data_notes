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

