# 第2周

[TOC]

## 四、多变量线性回归(Linear Regression with Multiple Variables)

### 4.1 多维特征

参考视频: 4 - 1 - Multiple Features (8 min).mkv

目前为止，我们探讨了单变量/特征的回归模型，现在我们对房价模型增加更多的特征，例如房间数楼层等，构成一个含有多个变量的模型，模型中的特征为$\left( {x_{1}},{x_{2}},...,{x_{n}} \right)$。

[![img](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/raw/master/images/591785837c95bca369021efa14a8bb1c.png)](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/blob/master/images/591785837c95bca369021efa14a8bb1c.png)

增添更多特征后，我们引入一系列新的注释：

$n$ 代表特征的数量

${x^{\left( i \right)}}$代表第 $i$ 个训练实例，是特征矩阵中的第$i$行，是一个**向量**（**vector**）。

比方说，上图的

${x}^{(2)}\text{=}\begin{bmatrix} 1416\\ 3\\ 2\\ 40 \end{bmatrix}$，

${x}_{j}^{\left( i \right)}$代表特征矩阵中第 $i$ 行的第 $j$ 个特征，也就是第 $i$ 个训练实例的第 $j$ 个特征。

如上图的$x_{2}^{\left( 2 \right)}=3,x_{3}^{\left( 2 \right)}=2$，

支持多变量的假设 $h$ 表示为：$h_{\theta}\left( x \right)={\theta_{0}}+{\theta_{1}}{x_{1}}+{\theta_{2}}{x_{2}}+...+{\theta_{n}}{x_{n}}$，

这个公式中有$n+1$个参数和$n$个变量，为了使得公式能够简化一些，引入$x_{0}=1$，则公式转化为：$h_{\theta} \left( x \right)={\theta_{0}}{x_{0}}+{\theta_{1}}{x_{1}}+{\theta_{2}}{x_{2}}+...+{\theta_{n}}{x_{n}}$

此时模型中的参数是一个$n+1$维的向量，任何一个训练实例也都是$n+1$维的向量，特征矩阵$X$的维度是 $m*(n+1)$。 因此公式可以简化为：$h_{\theta} \left( x \right)={\theta^{T}}X$，其中上标$T$代表矩阵转置。

### 4.2 多变量梯度下降

参考视频: 4 - 2 - Gradient Descent for Multiple Variables (5 min).mkv

与单变量线性回归类似，在多变量线性回归中，我们也构建一个代价函数，则这个代价函数是所有建模误差的平方和，即：$J\left( {\theta_{0}},{\theta_{1}}...{\theta_{n}} \right)=\frac{1}{2m}\sum\limits_{i=1}^{m}{{{\left( h_{\theta} \left({x}^{\left( i \right)} \right)-{y}^{\left( i \right)} \right)}^{2}}}$ ，

其中：$h_{\theta}\left( x \right)=\theta^{T}X={\theta_{0}}+{\theta_{1}}{x_{1}}+{\theta_{2}}{x_{2}}+...+{\theta_{n}}{x_{n}}$ ，

我们的目标和单变量线性回归问题中一样，是要找出使得代价函数最小的一系列参数。 多变量线性回归的批量梯度下降算法为：

[![img](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/raw/master/images/41797ceb7293b838a3125ba945624cf6.png)](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/blob/master/images/41797ceb7293b838a3125ba945624cf6.png)

即：

[![img](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/raw/master/images/6bdaff07783e37fcbb1f8765ca06b01b.png)](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/blob/master/images/6bdaff07783e37fcbb1f8765ca06b01b.png)

求导数后得到：

求导数后得到：

[![img](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/raw/master/images/dd33179ceccbd8b0b59a5ae698847049.png)](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/blob/master/images/dd33179ceccbd8b0b59a5ae698847049.png)

当$n>=1$时， ${{\theta }_{0}}:={{\theta }_{0}}-a\frac{1}{m}\sum\limits_{i=1}^{m}{({{h}_{\theta }}({{x}^{(i)}})-{{y}^{(i)}})}x_{0}^{(i)}$

${{\theta }_{1}}:={{\theta }_{1}}-a\frac{1}{m}\sum\limits_{i=1}^{m}{({{h}_{\theta }}({{x}^{(i)}})-{{y}^{(i)}})}x*{1}^{(i)}$

${{\theta }_{2}}:={{\theta }_{2}}-a\frac{1}{m}\sum\limits_{i=1}^{m}{({{h}_{\theta }}({{x}^{(i)}})-{{y}^{(i)}})}x*{2}^{(i)}$

我们开始随机选择一系列的参数值，计算所有的预测结果后，再给所有的参数一个新的值，如此循环直到收敛。

代码示例：

计算代价函数 $J\left( \theta \right)=\frac{1}{2m}\sum\limits_{i=1}^{m}{{{\left( {h_{\theta}}\left( {x^{(i)}} \right)-{y^{(i)}} \right)}^{2}}}$ 其中：${h_{\theta}}\left( x \right)={\theta^{T}}X={\theta_{0}}{x_{0}}+{\theta_{1}}{x_{1}}+{\theta_{2}}{x_{2}}+...+{\theta_{n}}{x_{n}}$

**Python** 代码：

```python
def computeCost(X, y, theta):
    inner = np.power(((X * theta.T) - y), 2)
    return np.sum(inner) / (2 * len(X))
```

### 4.3 梯度下降法实践1-特征缩放

以房价问题为例，假设我们使用两个特征，房屋的尺寸和房间的数量，尺寸的值为 0-2000平方英尺，而房间数量的值则是0-5，以两个参数分别为横纵坐标，绘制代价函数的等高线图能，看出图像会显得很扁，梯度下降算法需要非常多次的迭代才能收敛。

[![img](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/raw/master/images/966e5a9b00687678374b8221fdd33475.jpg)](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/blob/master/images/966e5a9b00687678374b8221fdd33475.jpg)

解决的方法是尝试将所有特征的尺度都尽量缩放到-1到1之间。如图

[![img](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/raw/master/images/b8167ff0926046e112acf789dba98057.png)](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/blob/master/images/b8167ff0926046e112acf789dba98057.png)

最简单的方法是令：${{x}_{n}}=\frac{{{x}_{n}}-{{\mu}_{n}}}{{{s}_{n}}}$，其中 ${\mu_{n}}$是平均值，${s_{n}}$是标准差，即将数据标准话。

### 4.4 梯度下降法实践2-学习率

参考视频: 4 - 4 - Gradient Descent in Practice II - Learning Rate (9 min).mkv

梯度下降算法收敛所需要的迭代次数根据模型的不同而不同，我们不能提前预知，我们可以绘制**迭代次数和代价函数的图表**来观测算法在何时趋于收敛。

[![img](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/raw/master/images/cd4e3df45c34f6a8e2bb7cd3a2849e6c.jpg)](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/blob/master/images/cd4e3df45c34f6a8e2bb7cd3a2849e6c.jpg)

也有一些自动测试是否收敛的方法，例如将代价函数的变化值与某个阀值（例如0.001）进行比较，但通常看上面这样的图表更好。

梯度下降算法的每次迭代受到学习率的影响，如果学习率$a$过小，则达到收敛所需的迭代次数会非常高；如果学习率$a$过大，每次迭代可能不会减小代价函数，可能会越过局部最小值导致无法收敛。

通常可以考虑尝试些学习率：

$\alpha=0.01，0.03，0.1，0.3，1，3，10$

### 4.5 特征和多项式回归

参考视频: 4 - 5 - Features and Polynomial Regression (8 min).mkv

如房价预测问题，

[![img](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/raw/master/images/8ffaa10ae1138f1873bc65e1e3657bd4.png)](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/blob/master/images/8ffaa10ae1138f1873bc65e1e3657bd4.png)

$h_{\theta}\left( x \right)={\theta_{0}}+{\theta_{1}}\times{frontage}+{\theta_{2}}\times{depth}$

${x_{1}}=frontage$（临街宽度），${x_{2}}=depth$（纵向深度），$x=frontage*depth=area$（面积），则：${h_{\theta}}\left( x \right)={\theta_{0}}+{\theta_{1}}x$。 线性回归并不适用于所有数据，有时我们需要曲线来适应我们的数据，比如一个二次方模型：$h_{\theta}\left( x \right)={\theta_{0}}+{\theta_{1}}{x_{1}}+{\theta_{2}}{x_{2}^2}$ 或者三次方模型： $h_{\theta}\left( x \right)={\theta_{0}}+{\theta_{1}}{x_{1}}+{\theta_{2}}{x_{2}^2}+{\theta_{3}}{x_{3}^3}$

[![img](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/raw/master/images/3a47e15258012b06b34d4e05fb3af2cf.jpg)](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/blob/master/images/3a47e15258012b06b34d4e05fb3af2cf.jpg)

通常我们需要**先观察数据然后再决定准备尝试怎样的模型**。 另外，我们可以令

${{x}_{2}}=x_{2}^{2},{{x}_{3}}=x_{3}^{3}$，从而将模型转化为线性回归模型。

根据函数图形特性，我们还可以使：

${{{h}}_{\theta}}(x)={{\theta }_{0}}\text{+}{{\theta }_{1}}(size)+{{\theta}_{2}}{{(size)}^{2}}$

或者:

${{{h}}_{\theta}}(x)={{\theta }_{0}}\text{+}{{\theta }_{1}}(size)+{{\theta }_{2}}\sqrt{size}$

注：如果我们采用多项式回归模型，在运行梯度下降算法前，特征缩放非常有必要。

什么情况下使用多项式线性回归？



### 4.6 正规方程

到目前为止，我们都在使用梯度下降算法，但是对于某些线性回归问题，正规方程方法是更好的解决方案。如：

[![img](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/raw/master/images/a47ec797d8a9c331e02ed90bca48a24b.png)](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/blob/master/images/a47ec797d8a9c331e02ed90bca48a24b.png)

正规方程是通过求解下面的方程来找出使得代价函数最小的参数的：$\frac{\partial}{\partial{\theta_{j}}}J\left( {\theta_{j}} \right)=0$ 。 假设我们的训练集特征矩阵为 $X$（包含了 ${{x}_{0}}=1$）并且我们的训练集结果为向量 $y$，则利用正规方程解出向量 $\theta ={{\left( {X^T}X \right)}^{-1}}{X^{T}}y$ 。 上标**T**代表矩阵转置，上标-1 代表矩阵的逆。设矩阵$A={X^{T}}X$，则：${{\left( {X^T}X \right)}^{-1}}={A^{-1}}$ 以下表示数据为例：

[![img](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/raw/master/images/261a11d6bce6690121f26ee369b9e9d1.png)](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/blob/master/images/261a11d6bce6690121f26ee369b9e9d1.png)

即：

[![img](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/raw/master/images/c8eedc42ed9feb21fac64e4de8d39a06.png)](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/blob/master/images/c8eedc42ed9feb21fac64e4de8d39a06.png)

运用正规方程方法求解参数：

[![img](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/raw/master/images/b62d24a1f709496a6d7c65f87464e911.jpg)](https://github.com/fengdu78/Coursera-ML-AndrewNg-Notes/blob/master/images/b62d24a1f709496a6d7c65f87464e911.jpg)

注：对于那些不可逆的矩阵（通常是因为特征之间不独立，如同时包含英尺为单位的尺寸和米为单位的尺寸两个特征，也有可能是特征数量大于训练集的数量），正规方程方法是不能用的。

梯度下降与正规方程的比较：

| 梯度下降                      | 正规方程                                                     |
| ----------------------------- | ------------------------------------------------------------ |
| 需要选择学习率$\alpha$        | 不需要                                                       |
| 需要多次迭代                  | 一次运算得出                                                 |
| 当特征数量$n$大时也能较好适用 | 需要计算${{\left( {{X}^{T}}X \right)}^{-1}}$ 如果特征数量n较大则运算代价大，因为矩阵逆的计算时间复杂度为$O\left( {{n}^{3}} \right)$，通常来说当$n$小于10000 时还是可以接受的 |
| 适用于各种类型的模型          | 只适用于线性模型，不适合逻辑回归模型等其他模型               |

总结一下，只要特征变量的数目并不大，标准方程是一个很好的计算参数$\theta $的替代方法。具体地说，只要**特征变量数量小于一万**，我通常使用标准方程法，而不使用梯度下降法。

随着我们要讲的学习算法越来越复杂，例如，当我们讲到分类算法，像逻辑回归算法，我们会看到，实际上对于那些算法，并不能使用标准方程法。对于那些更复杂的学习算法，我们将不得不仍然使用梯度下降法。因此，梯度下降法是一个非常有用的算法，可以用在有大量特征变量的线性回归问题。或者我们以后在课程中，会讲到的一些其他的算法，因为标准方程法不适合或者不能用在它们上。但对于这个特定的线性回归模型，标准方程法是一个比梯度下降法更快的替代算法。所以，根据具体的问题，以及你的特征变量的数量，这两种算法都是值得学习的。

正规方程的**python**实现：

```python
import numpy as np
    
 def normalEqn(X, y):
    
   theta = np.linalg.inv(X.T@X)@X.T@y #X.T@X等价于X.T.dot(X)
    
   return theta
```

