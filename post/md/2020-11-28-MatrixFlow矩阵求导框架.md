---
layout: post
title: 2020-11-28-MatrixFlow矩阵求导框架
date: 2020-11-28
categories: blog
tags: [linux,操作系统]
description: 文件系统
---

# 矩阵求导及一个简单的MatrixFlow矩阵自动求解梯度框架
pytorch是非常好用和容易上手的深度学习框架，因为它所构建的是动态图，极大地方便了编写代码和排查错误。本文将模仿Pytorch自动求解梯度，从原理和代码实现两方面讲解细节，与pytorch不同的是本文实现的框架底层是matrix而非tensor。原理方面参考网上的优秀文章非原创，MatrixFlow框架代码实现是原创的。在自动求解梯度框架的基础上封装一个前馈神经网络和一个循环神经网络。

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

解：先使用矩阵乘法、逐元素函数法则求微分:  

$$ df = \boldsymbol{a}^{T}(exp(\boldsymbol{Xb})\odot(d\boldsymbol{Xb})) $$ 

再套上迹并做交换：  

$$ df = tr(\boldsymbol{a}^{T}(exp(\boldsymbol{Xb})\odot (d\boldsymbol{Xb}))) $$ 

$$ df = tr((\boldsymbol{a} \odot exp(\boldsymbol{Xb}))^{T}d\boldsymbol{Xb}) $$ 

$$ df = tr(\boldsymbol{b}(\boldsymbol{a} \odot exp(\boldsymbol{Xb}))^{T} d\boldsymbol{X}) $$  

$$ df = tr(((\boldsymbol{a} \odot exp(\boldsymbol{Xb}))\boldsymbol{b}^{T})^{T}d\boldsymbol{X}) $$  


对照导数与微分的联系，  $ df = tr(\frac{\partial f}{\partial \boldsymbol{X}}d\boldsymbol{X}) $ ，得到：  

$$ \frac{\partial f}{\partial \boldsymbol{X}} = (\boldsymbol{a} \odot exp(\boldsymbol{Xb}))\boldsymbol{b}^{T} $$  

# 2. Pytorch计算图讲解

限于我的知识水平有限，而恰巧网上的一篇博客讲解的非常好，所以这里我放上大佬的博客链接，并且就简单介绍下计算图的一些概念。

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

![计算图](../images/20201128/1.png)

再举一个复杂的例子： 一个神经网络中有5个神经元a,b,c,d,L；其中w1~w4为权重矩阵，L为输出。满足以下计算关系：

```C
b = w1 ∗ a
c = w2 ∗ a
d = w3 ∗ b + w4 ∗ c
L = 10 − d
```

![例子2](../images/20201128/5.png)

求L对w1的偏导： $ \frac{dL}{dw_{1}} = \frac{dL}{dd} \frac{dd}{db} \frac{db}{dw1} $  

求L对a的偏导： $ \frac{dL}{da} = \frac{dL}{dd}\frac{dd}{db}\frac{db}{da} + \frac{dL}{dd}\frac{dd}{dc}\frac{dc}{da} $

实际上，pytorch计算L对w1的偏导时，正是沿着反向传播计算图的路径执行的，先计算 $ \frac{dL}{dd} $ 然后计算 $\frac{dL}{dd} \frac{dd}{db} $最后计算 $ \frac{dL}{dd} \frac{dd}{db}\frac{db}{dw_{1}} $

但是该文章实现的是MatrixFlow，底层的基本数据是矩阵而不是标量，链式法则并不适用（见1.4小节），所以这里需要进行修改，使用第1节的矩阵微分和迹技巧。  

我们欲求 $ \frac{L}{w_{1}} $ 则先求解 $ dL $ ，微分的求解是遵循链式法则的，然后根据微分与导数的关系算出导数。  

# 3 MatrixFlow自动求导的Python实现  

根据前面的讨论，本节我们自己手动完成一个矩阵自动求导的框架，并取名为MatrixFlow。这个矩阵求导框架底层是矩阵，并且对于任意一个标量需要表示成\[\[a\]\]这种形式。为了验证MatrixFlow的正确性，我们将会对比Pytorch的梯度结果和MatrixFlow的求导结果。最后，在我们的求导框架之上再封装一个DNN和一个RNN，并进行简单的深度学习训练。

## 自动求导的实现  

MatrixFlow的基本元素是矩阵，那么首先需要定义矩阵在底层的表示方式，它被称作为Node，相当于计算图中的一个结点。根据第2节的讨论，Node的数据类型被定义如下：

```Python
    def __init__(self, tensor, requires_grad=False):
        self.tensor = np.array(tensor)
        self.requires_grad = requires_grad
        self.grad = 0.0
        self.grad_fn = None
        self.is_leaf = True
        self.father = None
        self.lchild = None
        self.rchild = None
        self.param = ""
```

每一个Node结点被表示为左操作数、右操作数、值、操作符类型。tensor是Node结点表达式的值，requires_grad表示为是该表达式是否需要求解梯度，grad表示为该结点的梯度，grad_fn表示为求解梯度时的函数，它的输入是计算图前驱结点的梯度、左操作数、右操作数，输出是左操作数的梯度，右操作数的梯度。is_leaf判断结点是否是叶子结点，在求解梯度时，当遇到叶子结点则不继续往下求导，因为叶子结点的左右孩子是None。father是该计算图的前驱结点，该信息用于调试而不用于正式的计算，一般情况下loss函数前驱前驱结点是None。lchild是左操作数结点，rchild是右操作数结点，param用于调试信息，正式计算时用不到。  

紧接着需要定义Node的操作符，这里MatrixFlow的基本操作有加法、减法、乘法、负号、relu、转置、连接操作。

为了方便叙述，这里只给出Node的加法、relu、连接三个操作，其余的操作可以参见源代码。加法操作如下：  

```Python
    # 左侧调用的add
    def __add__(self, value):
        if isinstance(value, (list, np.ndarray)):
            value = Node(value)  # requires_grad=False

        n_tensor = self.tensor + value.tensor
        n_node = Node(n_tensor)
        n_node.requires_grad = self.requires_grad or value.requires_grad

        n_node.grad_fn = grad_add
        n_node.is_leaf = False
        n_node.lchild = self
        n_node.lchild.father = n_node
        n_node.rchild = value
        n_node.rchild.father = n_node

        n_node.param = "add"
        return n_node

```

前三行用来类型转换，Node可以与List、np.ndarray相互转换和计算。第五行用来计算该表达式的值，第六行生成一个Node结点，后面的属性修改依据则是按照第2节的讨论进行。requrires_grad取决于孩子结点的属性，grad_fn是用来求导的函数。  


relu操作如下：  

```python
    def relu(self):
        n_tensor = np.where(self.tensor > 0, self.tensor, 0)
        n_node = Node(n_tensor)

        n_node.requires_grad = self.requires_grad
        n_node.grad_fn = grad_relu
        n_node.is_leaf = False
        n_node.lchild = self
        n_node.lchild.father = n_node
        n_node.rchild = None

        n_node.param = "relu"
        return n_node
```

relu是一个单操作数的运算，其左孩子是原矩阵，右孩子是None，其运算过程是将元素<0的值等于0，其导数定义为grad_relu。  

连接操作：  
连接操作并不是矩阵的基本操作，因此它没有被定义到Node中，但是我们可以根据基本操作合成一个连接操作。举个例子，输入A, B，求解[A, B]可以表示成Y = A * [I|0] + B * [0|I] ，也就是说让矩阵A乘以一个常数矩阵，矩阵B乘以一个常数矩阵，然后相加即可得到。所以连接操作定义如下：  

```python
def concat(A: Node, B: Node, axis=1):               # 这里先做1维的concat,有空再扩展
    '''
    :param A: A矩阵
    :param B: B矩阵
    :return: 拼接后的[A,B]矩阵，因为底层是矩阵，那么
    拼接函数可以这样实现：Y = A * [I|0] + B * [0|I]
    '''

    if B is None:                                   # 默认
        return A
    if A is None:
        return B

    mask1 = np.zeros((A.tensor.shape[-1], A.tensor[-1]+B.tensor.shape[-1]))
    mask2 = np.zeros((B.tensor.shape[-1], B.tensor.shape[-1]+A.tensor.shape[-1]))

    for i in range(A.tensor.shape[-1]):
        mask1[i][i] = 1
    for j in range(B.tensor.shape[-1]):
        mask2[j+A.tensor.shape[-1]][j] = 1
    mask1 = Node(mask1)
    mask2 = Node(mask2)

    Y = A * mask1 + B * mask2
    return Y
```

代码中mask1和mask2就是两个常数矩阵，Y则是拼接后的矩阵。

### 梯度求解函数

根据第2节的矩阵求导方法，利用**微分的链式法则**（而不是导数的链式法则），结合矩阵微分法则和迹技巧可以求解出导数。考虑如下函数：  

$$ l = f(y); y = AB $$  

其中y和l是标量，f是标量函数，A、B是矩阵，那么求解 $ \frac{\partial y}{\partial A} $ 和 $ \frac{\partial y}{\partial B} $ 可以表示为：  

$$ dl = tr(\frac{\partial l}{\partial y}^{T}d(y)) \\ = tr(\frac{\partial l}{\partial y}^{T}d(AB)) \\ = tr(\frac{\partial l}{\partial y}^{T}(dA)B) + tr(\frac{\partial l}{\partial y}^{T}A(dB)) = \\ tr(B\frac{\partial l}{\partial y}^{T}dA) + tr(\frac{\partial l}{\partial y}^{T}AdB) $$  

对照导数与微分的关系，可以计算出：  

$$ \frac{\partial l}{\partial A} = \frac{\partial l}{\partial y}B^{T}, \frac{\partial l}{\partial B} = A^{T}\frac{\partial l}{\partial y} $$  

那么矩阵乘法的梯度函数可以得到如下：  

```python
def grad_mul(gradient, lchild, rchild):
    return np.matmul(gradient, rchild.tensor.T), np.matmul(lchild.tensor.T, gradient)
```

其中gradient代表上面的 $ \frac{\partial l}{\partial y} $ 。类似地，我们可以得到加法、减法、relu、转置的关于左、右操作数的导数。详细的导数函数请参考源代码。  

接下来是最重要的一个函数，backward()函数。backward()函数是一个递归函数，用来计算每一个结点的左右操作数的导数。该函数传入的参数是上式中类似于 $ \frac{\partial l}{\partial y} $ ，的变量，也就是上一步的导数，那么对于计算图的根节点（一般是loss函数），它必须传入一个[[1.0]]，这是因为 $ \frac{\partial l}{\partial y} = [[ 1.0]] $ .

在backward函数内，首先积累从上一步计算出来的梯度，然后如果是叶子结点，那么继续反向过程已到底，可以直接退出。如果不是叶子结点，则先分别计算左右两个操作数的导数，如果孩子结点不为空那么递归地反向传播孩子结点。

代码如下： 
```python
    def backward(self, gradient=np.ones((1, 1), dtype=np.float)):
        if not self.requires_grad:
            return

        self.grad += gradient
        if not self.is_leaf:
            left_grad, right_grad = self.grad_fn(gradient, self.lchild, self.rchild)
            if self.lchild is not None:
                if self.lchild.requires_grad:
                    self.lchild.backward(gradient=left_grad)

            if self.rchild is not None:
                if self.rchild.requires_grad:
                    self.rchild.backward(gradient=right_grad)
```

最后，还需要清空梯度的函数，该函数也是递归的：
```python
    def zeros_grad(self):
        self.grad = 0
        if self.lchild is not None:
            self.lchild.zeros_grad()
        if self.rchild is not None:
            self.rchild.zeros_grad()
        return
```

至此我们的MatrixFlow自动求导框架已经搭建完毕，接下来和pytorch对比求导结果，看看是否正确。

### 与pytorch结果对比  

我们创建一个前馈神经网络，然后对比各个参数的梯度，测试代码如下：  

```python
if __name__ == "__main__":
    # 简单测试与pytorch对比
    w1 = np.random.uniform(-1, 1, (2, 5))
    w2 = np.random.uniform(-1, 1, (5, 1))

    b1 = np.random.uniform(-1, 1, (1, 5))
    b2 = np.random.uniform(-1, 1, (1, 1))

    x = np.random.uniform(-3.14, 3.14, (100, 2))
    y = np.sin(x[:, 0]) + np.cos(x[:, 1])
    y = y.reshape((y.shape[0], 1))

    # node 过程
    W1 = Node(w1, requires_grad=True)
    W2 = Node(w2, requires_grad=True)
    B1 = Node(b1, requires_grad=True)
    B2 = Node(b2, requires_grad=True)

    hid = Node(x) * W1 + Node(np.ones((x.shape[0], 1))) * B1
    hid = hid.relu()
    out = hid * W2 + Node(np.ones((hid.tensor.shape[0], 1))) * B2
    loss = (Node(y) - out).T() * (Node(y) - out)
    loss.backward()

    # tensor 过程
    WW1 = torch.tensor(w1, requires_grad=True)
    WW2 = torch.tensor(w2, requires_grad=True)
    BB1 = torch.tensor(b1, requires_grad=True)
    BB2 = torch.tensor(b2, requires_grad=True)

    hhid = torch.mm(torch.tensor(x), WW1) + torch.mm(torch.tensor(np.ones((x.shape[0], 1))), BB1)
    hhid = hhid.relu()
    oout = torch.mm(hhid, WW2) + torch.mm(torch.tensor(np.ones((hhid.shape[0], 1))), BB2)
    lloss = torch.mm((torch.tensor(y) - oout).T , (torch.tensor(y) - oout))
    lloss.backward()

    print("Pytorch W1: ")
    print("{}".format(" ".join(str(e.data.float())for e in WW1[0])))
    print("{}".format(" ".join(str(e.data.float()) for e in WW1[1])))

    print("Node W1: ")
    print("{}".format(" ".join(str(e) for e in W1.tensor[0])))
    print("{}".format(" ".join(str(e) for e in W1.tensor[1])))

    print("Pytorch W2: ")
    print("{}".format(" ".join(str(e) for e in WW2)))

    print("Node W2: ")
    print("{}".format(" ".join(str(e) for e in W2.tensor)))

    print("Pytorch B1: ")
    print("{}".format(" ".join(str(e) for e in BB1)))

    print("Node B1: ")
    print("{}".format(" ".join(str(e) for e in B1.tensor)))

    print("Pytorch B2: ")
    print("{}".format(" ".join(str(e) for e in BB2)))

    print("Node B2: ")
    print("{}".format(" ".join(str(e) for e in B2.tensor)))

```  
测试结果如下：
![与python对照](/images/20201128-2.jpg)

可以看到，MatrixFlow与Pytorch得到的梯度是一样的。  

### 封装一个DNN  

```python
class MLPRegressor:
    def __init__(self, hidden_size:list, activate="relu"):
        self.hidden_size = hidden_size
        self.activate = activate
        self.weight = []
        self.bias = []

    def fit(self, x:np.ndarray, y:np.ndarray):
        x = Node(x)
        y = Node(y)
        self.hidden_size.insert(0, x.tensor.shape[-1])
        for i in range(len(self.hidden_size)):
            if i == len(self.hidden_size)-1:
                w = np.random.uniform(-1, 1, (self.hidden_size[i], 1))
                self.weight.append(Node(w, requires_grad=True))
                b = np.random.uniform(-1, 1, (1,1))
                self.bias.append(Node(b, requires_grad=True))
            else:
                w = np.random.uniform(-1, 1, (self.hidden_size[i], self.hidden_size[i+1]))
                self.weight.append(Node(w, requires_grad=True))
                b = np.random.uniform(-1, 1, (1, self.hidden_size[i+1]))
                self.bias.append(Node(b, requires_grad=True))

        for epoch in range(1000):
            for i in range(len(self.hidden_size)):
                if i == 0:
                    hid = x * self.weight[i] + Node(np.ones((x.tensor.shape[0], 1))) * self.bias[i]
                    hid = hid.relu()
                elif i == len(self.hidden_size) - 1:
                    out = hid * self.weight[i] + Node(np.ones((hid.tensor.shape[0], 1))) * self.bias[i]
                else:
                    hid = hid * self.weight[i] + Node(np.ones((hid.tensor.shape[0], 1))) * self.bias[i]
                    hid = hid.relu()

            loss = (y - out).T() * (y - out)
            loss.backward()

            for w in self.weight:
                w.tensor = w.tensor - 0.001 * w.grad
                w.zeros_grad()

            for b in self.bias:
                b.tensor = b.tensor - 0.001 * b.grad
                b.zeros_grad()
        return

    def predict(self, x:np.ndarray):
        x = Node(x)
        for i in range(len(self.hidden_size)):
            if i == 0:
                hid = x * self.weight[i] + Node(np.ones((x.tensor.shape[0], 1))) * self.bias[i]
                hid = hid.relu()
            elif i == len(self.hidden_size) - 1:
                out = hid * self.weight[i] + Node(np.ones((hid.tensor.shape[0], 1))) * self.bias[i]
            else:
                hid = hid * self.weight[i] + Node(np.ones((hid.tensor.shape[0], 1))) * self.bias[i]
                hid = hid.relu()
        return out
```  

程序运行结果：  
![DNN训练损失函数变化](../images/20201128/3.png)

从上图中的损失函数一直递减直到最后收敛可以知道，我们计算的自动求梯度框架MatirxFlow正确地计算出了梯度。

### 封装一个RNN  

```python

if __name__ == "__main__":
    测试MLP
    mlp = MLPRegressor([5, 3], activate="relu")
    x = np.random.uniform(-3.14, 3.14, (100, 2))
    y = np.sin(x[:, 0]) + np.cos(x[:, 1])
    y = y.reshape((y.shape[0], 1))
    
    mlp.fit(x, y)
    res = mlp.predict(x)
    print(y)
    print(res.tensor)
```

程序运行结果：

![RNN训练损失函数变化](../images/20201128/4.png)





