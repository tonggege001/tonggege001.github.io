# 矩阵求导及一个简单的MatrixFlow矩阵自动求解梯度框架
pytorch是非常好用和容易上手的深度学习框架，因为它所构建的是动态图，极大地方便了编写代码和排查错误。本文将模仿Pytorch自动求解梯度，从原理和代码实现两方面讲解细节，与pytorch不同的是本文实现的框架底层是matrix而非tensor。原理方面参（zhao）考（ban）网上的优秀文章，代码实现是原创的。在自动求解梯度框架的基础上封装一个前馈神经网络和一个循环神经网络。

# 1. 矩阵微分求导讲解  
知乎这个博主讲解矩阵求导非常好，本文矩阵微分讲解部分参考了他的。博客地址： https://zhuanlan.zhihu.com/p/24709748

矩阵微分是多变量函数微分的推广。矩阵微分（包括矩阵偏导和梯度）是矩阵的重要运算工具之一，在统计学、流行计算、微分几何、计量经济、机器学习等方面有着广泛的应用。在许多工程应用中，参数往往都表示成向量或者据很的形式，对矩阵或者向量求导则变得尤其重要。  

## 1.1 符号定义  
首先对变元和函数做统一的定义：  
$ \boldsymbol{x} = [x_{1}, ... , x_{m}]^{T} \in \boldsymbol{R}^{m} $ 为实变量变元  
$ \boldsymbol{X} = [\boldsymbol{x_{1}}, ... ,\boldsymbol{x_{n}}] \in \boldsymbol{R}^{m \times n} $ 为实矩阵变元 
$ f(\boldsymbol{x}) \in R $ 为实值标量函数，变元为 $ m \times 1 $ 实值向量 $ \boldsymbol{x} $ 
$ f(\boldsymbol{X}) \in R $ 为实值标量函数，变元为 $ m \times N $ 实值矩阵 $ \boldsymbol{X} $ 

向量函数和矩阵函数在此不介绍，有兴趣的可以参考《矩阵分析与应用》 张贤达著。
## 1.2 梯度矩阵的表示方式  

$$D_{\boldsymbol{X}}f(\boldsymbol{X}) = \frac{\partial f(\boldsymbol{X})}{\partial \boldsymbol{X^{T}}} = \left[\begin{matrix} \frac{\partial f(\boldsymbol{X})}{x_{1,1}} & \frac{\partial f(\boldsymbol{X})}{x_{2,1}} & ... & \frac{\partial f(\boldsymbol{X})}{x_{m,1}} \\ \frac{\partial f(\boldsymbol{X})}{x_{1,2}} & \frac{\partial f(\boldsymbol{X})}{x_{2,2}} & ...  & \frac{\partial f(\boldsymbol{X})}{x_{m,2}} \\ ... & ... & ... & ... \\ \frac{\partial f(\boldsymbol{X})}{x_{1,n}} & \frac{\partial f(\boldsymbol{X})}{x_{2,n}} & ... & \frac{\partial f(\boldsymbol{X})}{x_{m,n}}  \end{matrix} \right]  \;\;\;\;\;\;\;\;\tag{1}$$

$$\nabla_{\boldsymbol{x}}f(\boldsymbol{x}) = \frac{\partial f(\boldsymbol{X})}{\partial \boldsymbol{X}} = \left[\begin{matrix} \frac{\partial f(\boldsymbol{X})}{x_{1,1}} & \frac{\partial f(\boldsymbol{X})}{x_{1,2}} & ... & \frac{\partial f(\boldsymbol{X})}{x_{1,n}} \\ \frac{\partial f(\boldsymbol{X})}{x_{2,1}} & \frac{\partial f(\boldsymbol{X})}{x_{2,2}} & ...  & \frac{\partial f(\boldsymbol{X})}{x_{2,n}} \\ ... & ... & ... & ... \\ \frac{\partial f(\boldsymbol{X})}{x_{m,1}} & \frac{\partial f(\boldsymbol{X})}{x_{m,2}} & ... & \frac{\partial f(\boldsymbol{X})}{x_{m,n}}  \end{matrix} \right]  \;\;\;\;\;\;\;\;\tag{2}$$

上述（1）式是Jacobian 矩阵，（2）式是梯度矩阵，两者都是求解微分后的运算结果，它们相差一个转置。在流行计算、几何物理中Jacobian表示方式较多，而在优化领域、机器学习方面梯度矩阵用到的比较多。本节及后面所有小节均采用梯度矩阵方式。  

可以看到，标量对矩阵的导数实际上就是标量函数 f 对 矩阵 X 逐元素求导，然后将逐元素求导结果按照X的尺寸排列起来。但是这样做并不好，因为逐元素求导破坏了函数变元的**整体性**。所以需要寻求一个类似于单变量求导一样不破坏变元完整性的求导方法。  

## 1.3 矩阵微分和迹方法求导

在一元微积分中的导数与微分的关系有 $ df = f^{'}(x)dx $ ，多元微积分的梯度（标量与向量的导数）也与微分有联系（全微分公式）：  

$$ df = \sum_{i=1}^{n}\frac{\partial f}{\partial x_{i}}dx_{i} = (\frac{\partial f}{\partial \boldsymbol{x}})^{T} \tag{3}$$  

上式可以看成逐元素求导的变元与微分矩阵在同位置相乘的形式，那么可以得到矩阵函数微分：  

$$ df = \sum_{i=1}^{m}\sum_{j=1}^{n}\frac{\partial f}{\partial X_{i,j}}dX_{i,j}  = tr(\frac{\partial f }{\partial \boldsymbol{X}}^{T}d\boldsymbol{X}) \tag{4} $$

其中tr代表迹(trace)是方阵对角线元素之和，满足性质：对尺寸相同的矩阵 $ \boldsymbol{A}, \boldsymbol{B} $ 有 $ tr(\boldsymbol{A}^{T}\boldsymbol{B} = \sum_{i,j}A_{i,j}B_{i,j}) $
即 $ tr(\boldsymbol{A}^{T}\boldsymbol{B}) $  是矩阵A，B的内积。与梯度相似，第一个等号是全微分，第二个等号表达了矩阵导数与微分的联系：全微分 $ df $ 是导数 $ \frac{\partial f}{\partial \boldsymbol{X}} $ 与微分矩阵 $ d\boldsymbol{X} $ 的内积。  


## 1.4 建立微分运算法则  

1. 加减法： $ d(\boldsymbol{X} \pm \boldsymbol{Y}) = d\boldsymbol{X} \pm d\boldsymbol{Y} $ ； 矩阵乘法： $ d(\boldsymbol{XY}) = (d\boldsymbol{X})\boldsymbol{Y} + \boldsymbol{X}(d\boldsymbol{Y}) $ ； 转置： $ d(X^{T}) = (dX)^T $ ； 迹： $ dtr(X) = tr(dX) $ 

2. 逆： $ dX^{-1} = -X^{-1}dXX^{-1} $ 。此式可在 $ XX^{-1}=I $ 两侧求微分来证明。  

3. 行列式： $ d\|X\| = tr(X^{\star}dX) $ ，其中 $ X^{\star} $ 表示 $ X $ 的伴随矩阵，在X可逆时又可写作： $ d\|X\| = \|X\|tr(X^{-1}dX) $ 。此式可用Laplace展开来证明，详见张贤达《矩阵分析与应用》第279页。  

4. 逐元素乘法： $ d(X \odot Y) = dX \odot Y + X \odot dY $ ， $ \odot $ 表示尺寸相同的矩阵X，Y逐元素相乘。

5. 逐元素函数： $ d\sigma(X) = \sigma^{'}(X) \odot dX $ ， $ \sigma(X) = [\sigma (X_{i,j})] $ 是逐元素标量函数运算， $ \sigma^{'}=[\sigma^{'}(X_{i,j})] $ 是逐元素求导。  

我们试图利用矩阵导数与微分的联系：  

$$ df = tr((\frac{\partial f}{\partial X})^{T}dX) $$  

，在求出微分 $ df $ 后，表达式 $ \frac{\partial f}{\partial X} $ 就是所求的 $ X $ 的导数。那么该如何写成右侧的形式并得到导数呢？这里需要使用迹技巧：  

1. 标量套上迹： $ a=tr(a) $  

2. 转置： $ tr(A^{T}) = tr(A) $  

3. 线性： $ tr(A \pm B) = tr(A) \pm tr(B) $  。  

4. 矩阵乘法交换： $ tr(AB) = tr(BA) $  ，其中 $ A $ 与  $ B^{t} $ 尺寸相同。两侧都等于 $ \sum_{i,j}A_{i,j}B_{j,i} $  。 $ tr(A \odot B) = tr(B \odot A) $ ，其中A和B的形状一样。

5. 矩阵乘法/逐元素乘法交换： $ tr(A^{T}(B \odot C)) = tr((A \odot B)^{T}C) $


观察一下可以断言，**若标量函数f是矩阵X经加减乘法、逆、行列式、逐元素函数等运算构成，则使用相应的运算法则对f求微分，再使用迹技巧给df套上迹并将其它项交换至dX左侧，对照导数与微分的联系** $ df=tr((\frac{\partial f}{\partial X})^{T}dX) $ **，即能得到导数。**

**特别地，若矩阵退化为向量，对照导数与微分的联系** $ df = \frac{\partial f}{\partial \boldsymbol{x}}^{T}d\boldsymbol{x} $ ，即能得到导数。

在建立法则的最后，来谈一谈复合：假设已求得 $ \frac{\partial f}{\partial \boldsymbol{Y}} $ ，而Y是X的函数，如何求 $ \frac{\partial f}{\partial \boldsymbol{X}} $ 呢？在微积分中有标量的求导的链式法则 $ \frac{\partial f}{\partial x} = \frac{\partial f}{\partial y} \frac{\partial y}{\partial x} $ 但这里我们**不能随意沿用标量的链式法则**，因为矩阵对矩阵的导数 $ \frac{\partial \boldsymbol{Y}}{\partial \boldsymbol{X}} $ 是未定义的。（其实还是能够求导的，其方法是向量化变元X和Y，然后进行求导。详情请参考《矩阵分析与应用》张贤达著）于是我们继续追本溯源，链式法则是从何而来？源头仍然是微分。我们直接从微分入手建立复合法则：先写出 $ df = tr((\frac{\partial f}{\partial \boldsymbol{Y}})^{T}d\boldsymbol{Y}) $ ，再将 $ d\boldsymbol{Y} $ 用 $ d\boldsymbol{X} $ 表示出来代入，并使用迹技巧将其他项交换至dX左侧，即可得到 $ \frac{\partial f}{\partial \boldsymbol{X}} $  

## 1.5 矩阵微分的例子  

$ f=\boldsymbol{a}^{T}exp(\boldsymbol{Xb}) $ ，求 $ \frac{\partial f}{\partial \boldsymbol{X}} $ 。其中 $ \boldsymbol{a} $ 是 $ m \times 1 $ 列向量， $ \boldsymbol{X} $ 是 $ m \times n $ 矩阵， $ \boldsymbol{b} $ 是 $ n \times 1 $ 列向量，exp表示逐元素求指数，f是标量。  

解：先使用矩阵乘法、逐元素函数法则求微分： 
$$ df = \boldsymbol{a}^{T}(exp(\boldsymbol{Xb})\odot(d\boldsymbol{Xb})) $$  

再套上迹并做交换：  

$$ df = tr(\boldsymbol{a}^{T}(exp(\boldsymbol{Xb})\odot (d\boldsymbol{Xb}))) $$ 
$$ df = tr((\boldsymbol{a} \odot exp(\boldsymbol{Xb}))^{T}d\boldsymbol{Xb}) $$ 
$$ df = tr(\boldsymbol{b}(\boldsymbol{a} \odot exp(\boldsymbol{Xb}))^{T} d\boldsymbol{X})$$  
$$ df = tr(((\boldsymbol{a} \odot exp(\boldsymbol{Xb}))\boldsymbol{b}^{T})^{T}d\boldsymbol{X}) $$  

对照导数与微分的联系，  $ df = tr(\frac{\partial f}{\partial \boldsymbol{X}}d\boldsymbol{X}) $ ，得到：  

$$ \frac{\partial f}{\partial \boldsymbol{X}} = (\boldsymbol{a} \odot exp(\boldsymbol{Xb}))\boldsymbol{b}^{T} $$  

# 2. Pytorch计算图讲解
限于我的知识水平有限，语言水平匮乏，而恰巧网上的一篇博客讲解的非常好，所以这里我放上大佬的博客链接，并且就简单介绍下计算图的一些概念。

https://zhuanlan.zhihu.com/p/145353262  

## 2.1 前向传播  

前向传播就是构建一组运算顺序，在神经网络中就是从输入层 -> 隐藏层 -> 输出层 -> loss 的一组计算过程。因为表达式的计算过程是单向的并且不可颠倒的，所以这一组计算过程是单向的，所以我们称数据流向的方向是前向，这样的传播过程是前向传播。  

## 2.2 反向传播  
反向传播是从数据的顶层依次对用到的变量进行求解梯度的过程，根据链式法则，最先参与计算的操作数最后被求解梯度，因为这是一个嵌套多元函数，最外层函数的梯度最先被计算出来。  

## 2.3 计算图  

根据前向传播的过程，可以生成一个计算图，图的结点代表一次运算，图的遍表示数据的流入方向。前向传播的过程是单向的，所以这个计算图是一个有向无环图。

举个例子，对于如下计算：  

```C
x = 1
y = 2
z = ( y + x ) * x
```  

可生成如下的计算图：  

（insert图片）

再举一个复杂的例子： 一个神经网络中有5个神经元a,b,c,d,L；其中w1~w4为权重矩阵，L为输出。满足以下计算关系：

```C
b = w1 ∗ a
c = w2 ∗ a
d = w3 ∗ b + w4 ∗ c
L = 10 − d
```

（insert 图片） 

求L对w1的偏导： $ \frac{dL}{dw_{1}} = \frac{dL}{dd} \frac{dd}{db} \frac{db}{dw1} $  

求L对a的偏导： $ \frac{dL}{da} = \frac{dL}{dd}\frac{dd}{db}\frac{db}{da} + \frac{dL}{dd}\frac{dd}{dc}\frac{dc}{da} $

实际上，pytorch计算L对w1的偏导时，正是沿着反向传播计算图的路径执行的，先计算 $ \frac{dL}{dd} $ 然后计算 $\frac{dL}{dd} \frac{dd}{db} $最后计算 $ \frac{dL}{dd} \frac{dd}{db}\frac{db}{dw_{1}} $


但是该文章实现的是MatrixFlow，底层的基本数据是矩阵而不是标量，链式法则并不适用（见1.4小节），所以这里需要进行修改，使用第1节的矩阵微分和迹技巧。  

我们欲求 $ \frac{L}{w_{1}} $ 则先求解 $ dL $ ，微分的求解是遵循链式法则的，然后根据微分与导数的关系算出导数。  

# 3 自动求导的Python实现  






