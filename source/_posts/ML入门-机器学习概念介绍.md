---
title: ML入门-机器学习概念介绍
date: 2023-12-05 16:15:25
tags:
- ml
categories:
- Machine Learning
---
## 什么是机器学习
考虑传统的构建应用的方式，如下图所示：
![](https://developers.google.com/static/codelabs/tensorflow-1-helloworld/img/c72f871306134e45.png?hl=zh-cn)
我们定义规则，并使用一种编程语言表达规则，这些规则根据数据执行操作，我们的程序会返回确定的答案。
通过机器学习构建应用的方式过程非常相似，只是轴线不同：
![](https://developers.google.com/static/codelabs/tensorflow-1-helloworld/img/9b85a337ee816e1b.png?hl=zh-cn)
与传统的构建方式不同，我们不再需要定义规则并且也不需要用编程语言表达规则，我们只需提供答案（标签）以及数据，机器即可推断出用户确定答案与数据之间关系的规则。
例如，考虑到一个活动检测的场景，我们需要判断活动的类型：
![](https://developers.google.com/static/codelabs/tensorflow-1-helloworld/img/6ff58697a85931f4.png?hl=zh-cn)
我们需要收集大量的数据，并有效地标记其为“WALKING”或者“RUNNING”。然后，计算机可以根据数据推断出决定某一特定活动的不同模式的规则。
在传统编程中，代码被编译为通常称为程序的二进制文件。在机器学习中，通过数据和标签创建的项称为模型。因此我们可以把通过机器学习构建应用的过程表示为：
![](https://developers.google.com/static/codelabs/tensorflow-1-helloworld/img/693430bb4d7fa001.png?hl=zh-cn)
我们向模型传递一些数据，然后模型会使用从训练中推断的规则进行预测.

## 构建模型
以下是一组数据示例，我们该如何找出X和Y之间的关系？
<table><tbody><tr><td colspan="1" rowspan="1"><p>X:</p></td><td colspan="1" rowspan="1"><p>-1</p></td><td colspan="1" rowspan="1"><p>0</p></td><td colspan="1" rowspan="1"><p>1</p></td><td colspan="1" rowspan="1"><p>2</p></td><td colspan="1" rowspan="1"><p>3</p></td><td colspan="1" rowspan="1"><p>4</p></td></tr><tr><td colspan="1" rowspan="1"><p>Y:</p></td><td colspan="1" rowspan="1"><p>-2</p></td><td colspan="1" rowspan="1"><p>1</p></td><td colspan="1" rowspan="1"><p>4</p></td><td colspan="1" rowspan="1"><p>7</p></td><td colspan="1" rowspan="1"><p>10</p></td><td colspan="1" rowspan="1"><p>13</p></td></tr></tbody></table>
现在，我们来看一下执行该操作的代码，创建一个python文件。

### 使用pip安装tensorflow
```shell
pip install --upgrade tensorflow
##验证安装效果
python -c "import tensorflow as tf;print(tf. reduce_sum(tf.random.normal([1000, 1000])))"
```
> TensorFlow是一个由Google开发的开源机器学习框架，它可以用于构建和训练各种机器学习模型，包括神经网络。TensorFlow使用数据流图来表示数学运算，其中节点表示操作，边表示数据流动。它提供了一个灵活的平台，可以在各种不同的硬件上运行，包括CPU、GPU和TPU。TensorFlow已经成为机器学习领域中最受欢迎的框架之一，被广泛应用于深度学习、自然语言处理、计算机视觉和许多其他领域。
### 导入依赖
```python
import tensorflow as tf
import numpy as np
from tensorflow import keras
```
- numpy:该库可以轻松而快速地以列表形式展示您的数据。
- keras: tensorflow提供的将神经网络定义为一组顺序层的框架称为。

### 创建并编译神经网络
接下来，创建尽可能最简单的神经网络。该神经网络有一个层，该层有一个神经元，其输入形状只有一个值。

```python
model = tf.keras.Sequential([keras.layers.Dense(units=1, input_shape=[1])])
```

接下来，编写代码以编译神经网络。执行此操作时，您需要指定两个函数：`loss` 和 `optimizer`。

神经网络模型会进行猜测X与Y之间的关系，例如是 Y=10X+10。`loss` 函数将猜测答案与已知正确答案进行对比，并衡量结果的好坏。

接下来，模型使用 `optimizer` 函数进行另一个猜测。它会根据损失函数的结果，最大限度地减少损失。此时，可能会得到 Y=5X+5 之类的内容。虽然这样仍有些糟糕，但更接近正确的结果（损失更低）。

模型会针对即将经历的周期执行同样的操作。

首先，以下方法将告知模型使用 `mean_squared_error` 表示损失，以及随机梯度下降法 **(**`sgd`**)** 作为优化器。

```
model.compile(optimizer='sgd', loss='mean_squared_error')
```

### 准备数据
我们使用numpy的python库用来将上表中的数据转化为模型的训练数据，在 numpy 中使用 `np.array[]` 将值指定为数组:
```python
xs = np.array([-1.0, 0.0, 1.0, 2.0, 3.0, 4.0], dtype=float)
ys = np.array([-2.0, 1.0, 4.0, 7.0, 10.0, 13.0], dtype=float)
```
现在，我们已经具备了定义神经网络所需的全部代码。下一步是训练该代码，查看它能否推断出这些数字之间的模式，并使用其创建模型。

### 训练神经网络
训练神经网络的过程是在 model.fit 调用中的，神经网络从其中了解 X 和 Y 之间的关系。在这种情况下，它会在进行猜测、衡量好坏（损失），或通过优化器做出另一个猜测之前遍历整个循环。它会按照您指定的周期数运行。运行该代码时，您会看到每个周期输出的损失:
```python
model.fit(xs, ys, epochs=500)
```
输出如下所示，随着训练的进行，损失会很快减小：
![](https://developers.google.com/static/codelabs/tensorflow-1-helloworld/img/81ca5e71298b414b.png?hl=zh-cn)

### 使用模型
模型经过训练后已经了解了`X` 和 `Y` 之间的关系（当然不一定准确）。可以使用 model.predict 方法求出先前未知 X 的 Y。例如，如果 `X` 为 10，您认为 `Y` 为多少？
```python
print(model.predict([10.0]))
##输出类似于：[[30.99968]]
```
通过观察上面表格的数据，我们可以知道X和Y的关系为：Y=3X+1。因此当`X`=10时，`Y`应该为31。但模型的输出不一定为31。神经网络处理概率，因此它计算出 X 和 Y 之间的关系有很高的概率为 Y=3X+1，但仅凭 6 个数据点不可能确定其关系。所得结果接近 31，但不一定是 31。