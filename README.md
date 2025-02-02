# PyDyNet：Neuron Network(DNN, CNN, RNN, etc) implementation using Numpy based on Autodiff

前作：[PyNet: Use NumPy to build neuron network](https://github.com/Kaslanarian/PyNet)。在那里我们基于求导规则实现了全连接网络。在这里，我们向当今的深度学习框架看齐，实现属于自己的DL框架。

## Update

- 5.10:修改损失函数的定义方式：加入reduction机制，加入Embedding;
- 5.15:重构了RNN, LSTM和GRU，支持双向;
- 5.16:允许PyDyNet作为第三方库安装；开始手册的撰写(基于Sphinx).
- ...

## Overview

PyDyNet也是纯NumPy实现的神经网络，语法受PyTorch的启发，大致结构如下：

```mermaid
graph BT
	N ----> Data(DataLoader)
	N(numpy.ndarray) --> A(Tensor)
	A --Eager execution--> B(Basic operators: add, exp, etc)
	B --> E(Mechanism: Dropout, BN, etc)
	E --> D
	B --> C(Complex operators: softmax, etc)
	C --> E
	B --> D(Base Module:Linear, Conv2d, etc)
	C --> D
	B --Autograd--> A
	N ----> GD(Optimizer:SGD, Adam, etc)
	D --> M(Module:DNN, CNN, RNN, etc)
	M --> Mission(PyDyNet)
	Data --> Mission
	GD --> Mission
```

我们实现了：

1. 将NumPy数组包装成具有梯度等信息的张量(Tensor):

   ```python
   from pydynet import Tensor

   x = Tensor(1., requires_grad=True)
   print(x.data) # 1.
   print(x.ndim, x.shape, x.is_leaf) # 0, (), True
   ```

2. 将NumPy数组的计算(包括数学运算、切片、形状变换等)抽象成基础算子(Basic operators)，并对部分运算加以重载：

   ```python
   from pydynet import Tensor
   import pydynet.functional as F

   x = Tensor([1, 2, 3])
   y = F.exp(x) + x
   z = F.sum(x)
   print(z.data) # 36.192...
   ```

3. 手动编写基础算子的梯度，实现和PyTorch相同的动态图自动微分机制(Autograd)，从而实现反向传播

   ```python
   from pydynet import Tensor
   import pydynet.functional as F

   x = Tensor([1, 2, 3], requires_grad=True)
   y = F.log(x) + x
   z = F.sum(y)

   z.backward()
   print(x.grad) # [2., 1.5, 1.33333333]
   ```

4. 基于基础算子实现更高级的算子(Complex operators)，它们不再需要手动编写导数：

   ```python
   def simple_sigmoid(x: Tensor):
       return 1 / (1 + exp(-x))
   ```

5. 实现了Mudule，包括激活函数，损失函数等，从而我们可以像下面这样定义神经网络，损失函数项：

   ```python
   import pydynet.nn as nn
   import pydynet.functional as F

   n_input = 64
   n_hidden = 128
   n_output = 10

   class Net(nn.Module):
       def __init__(self) -> None:
           super().__init__()
           self.fc1 = nn.Linear(n_input, n_hidden)
           self.fc2 = nn.Linear(n_hidden, n_output)

       def forward(self, x):
           x = self.fc1(x)
           x = F.sigmoid(x)
           return self.fc2(x)

   net = Net()
   loss = nn.CrossEntropyLoss()
   l = loss(net(X), y)
   l.backward()
   ```

6. 实现了多种优化器(`optimizer.py`)，以及数据分批的接口(`dataloader.py`)，从而实现神经网络的训练；其中优化器和PyTorch一样支持权值衰减，即正则化；
7. Dropout机制，Batch Normalization机制，以及将网络划分成训练阶段和评估阶段；
8. 基于im2col高效实现Conv1d, Conv2d, max_pool1d和max_pool2d，从而实现CNN；
9. 支持多层的**双向**RNN，LSTM和GRU。

## Install

```bash
git clone https://github.com/Kaslanarian/PyDyNet
cd PyDyNet
python setup.py install
```

安装成功后就可以跑下面的例子

## Example

examples中是一些例子。

### AutoDiff

[autodiff.py](examples/autodiff.py)利用自动微分，对一个凸函数进行梯度下降：

![ad](src/autodiff.png)

### DNN

[DNN.py](examples/DNN.py)使用全连接网络对`sklearn`提供的数字数据集进行分类，训练参数

- 网络结构：Linear(64->64) + Sigmoid + Linear(64->10)；
- 损失函数：Cross Entropy Loss；
- 优化器：Adam(lr=0.01)；
- 训练轮次：50；
- 批大小(Batch size)：32.

训练损失，训练准确率和测试准确率：

<img src="src/DNN.png" alt="dnn" style="zoom:67%;" />

### CNN

[CNN.py](example/CNN.py)使用三种网络对`fetch_olivetti_faces`人脸(64×64)数据集进行分类并进行性能对比：

1. Linear + Sigmoid + Linear;
2. Conv1d + MaxPool1d + Linear + ReLU + Linear;
3. Conv2d + MaxPool2d + Linear + ReLU + Linear.

其余参数相同：

- 损失函数：Cross Entropy Loss；
- 优化器：Adam(lr=0.01)；
- 训练轮次：50；
- 批大小(Batch size)：32.

学习效果对比：

<img src="src/CNN.png" alt="cnn" style="zoom:67%;" />

## Droput & BN

[dropout_BN.py](example/dropout_BN.py)使用三种网络对`fetch_olivetti_faces`人脸(64×64)数据集进行分类并进行性能对比：

1. Linear + Sigmoid + Linear;
2. Linear + Dropout(0.05) + Sigmoid + Linear;
3. Linear + BN + Sigmoid + Linear.

其余参数相同：

- 损失函数：Cross Entropy Loss；
- 优化器：Adam(lr=0.01)；
- 训练轮次：50；
- 批大小(Batch size)：32.

学习效果对比：

<img src="src/dropout_BN.png" alt="BN" style="zoom:67%;" />

## RNN

[RNN.py](examples/RNN.py)中是一个用双向单层GRU对`sklearn`的数字图片数据集进行分类：

<img src="src/RNN.png" alt="RNN" style="zoom:67%;" />

目前RNN部分的模块还在开发中，包括扩展到双向（已完成），引入Attention机制等。
