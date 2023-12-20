---
title: ML入门-使用TensorFlow构建计算机视觉模型
date: 2023-12-05 17:11:24
tags:
- ml
categories:
- Machine Learning
---
## 目标
创建一个简单的深度神经网络（DNN），训练神经网络识别衣物
## 数据准备
我们使用名为 [Fashion MNIST](https://github.com/zalandoresearch/fashion-mnist) 的常见数据集中识别衣物。该数据集包含 70000 种衣物，分为 10 个不同的类别。每件衣物都采用 28x28 的灰度图像。下面是一些示例：

![](https://developers.google.com/static/codelabs/tensorflow-2-computervision/img/tenseimg10.png?hl=zh-cn)

与数据集关联的标签如下：

<table><tbody><tr><td colspan="1" rowspan="1"><p><strong>标签</strong></p></td><td colspan="1" rowspan="1"><p><strong>说明</strong></p></td></tr><tr><td colspan="1" rowspan="1"><p>0</p></td><td colspan="1" rowspan="1"><p>T 恤衫/上衣</p></td></tr><tr><td colspan="1" rowspan="1"><p>1</p></td><td colspan="1" rowspan="1"><p>裤子</p></td></tr><tr><td colspan="1" rowspan="1"><p>2</p></td><td colspan="1" rowspan="1"><p>套衫</p></td></tr><tr><td colspan="1" rowspan="1"><p>3</p></td><td colspan="1" rowspan="1"><p>裙子</p></td></tr><tr><td colspan="1" rowspan="1"><p>4</p></td><td colspan="1" rowspan="1"><p>外套</p></td></tr><tr><td colspan="1" rowspan="1"><p>5</p></td><td colspan="1" rowspan="1"><p>凉鞋</p></td></tr><tr><td colspan="1" rowspan="1"><p>6</p></td><td colspan="1" rowspan="1"><p>衬衫</p></td></tr><tr><td colspan="1" rowspan="1"><p>7</p></td><td colspan="1" rowspan="1"><p>运动鞋</p></td></tr><tr><td colspan="1" rowspan="1"><p>8</p></td><td colspan="1" rowspan="1"><p>包</p></td></tr><tr><td colspan="1" rowspan="1"><p>9</p></td><td colspan="1" rowspan="1"><p>踝靴</p></td></tr></tbody></table>

首先导入Tensorflow
```python
import tensorflow as tf
print(tf.__version__)
```
获取Fashion MNIST数据集
```python
mnist = tf.keras.datasets.fashion_mnist
```
在该对象上调用 load_data 会生成两组列表，分别是训练值(training_images)和测试值(training_labels)，它们分别表示衣物及其标签。test_images用作测试集，用来评价模型在未遇到的数据集上的泛化效果。

```python
(training_images, training_labels), (test_images, test_labels) = mnist.load_data()
```

使用pyplot查看数据示例
```python
import matplotlib.pyplot as plt
plt.imshow(training_images[0])
print(training_labels[0])
print(training_images[0])
```
其输出如下所示：
![](https://developers.google.com/static/codelabs/tensorflow-2-computervision/img/tenseimg20.png?hl=zh-cn)

所有值都是 0 到 255 之间的整数。在训练神经网络时，所有值在 0 到 1 之间处理起来更轻松，此过程称为归一化。幸运的是，Python 提供了一种无需循环便可归一化列表的简单方法

## 设计模型

```python
model = tf.keras.models.Sequential([tf.keras.layers.Flatten(),
                                    tf.keras.layers.Dense(128, activation=tf.nn.relu),
                                    tf.keras.layers.Dense(10, activation=tf.nn.softmax)])
```
这里准备了一个三层的模型。
-   `Sequential` 定义了神经网络中的层序列。
-   `Flatten` 会接受一个正方形并将其转换为一维矢量。
-   `Dense` 会添加一层神经元。
-   `Activation` 函数会告知各层神经元要执行的操作。选项有很多，但目前只采用以下选项：
-   `Relu` 实际上意味着，如果 X 大于 0，则返回 X，否则返回 0。它只会将 0 或更大的值传递到网络中的下一层。
-   `Softmax` 会接受一组值，并能有效地选择最大的值。例如，如果最后一层的输出是 \[0.1, 0.1, 0.05, 0.1, 9.5, 0.1, 0.05, 0.05, 0.05\]，那么您无需通过排序来获取最大值，它会返回 \[0,0,0,0,1,0,0,0,0\]。

## 编译和训练模型

现在已定义了模型，接下来要做的是构建模型。我们来创建模型，先使用 optimizer 和 loss 函数编译模型，然后使用训练数据和标签训练模型。

请注意，我们使用了 metrics= 参数，这样的话，TensorFlow 可根据已知答案（标签）检查预测结果，从而报告训练准确率。

```python
model.compile(optimizer = tf.keras.optimizers.Adam(),
              loss = 'sparse_categorical_crossentropy',
              metrics=['accuracy'])

model.fit(training_images, training_labels, epochs=5)
```
model.fit 执行时，您会看到损失和准确率：
```shell
Epoch 1/5
60000/60000 [=======] - 6s 101us/sample - loss: 0.4964 - acc: 0.8247
Epoch 2/5
60000/60000 [=======] - 5s 86us/sample - loss: 0.3720 - acc: 0.8656
Epoch 3/5
60000/60000 [=======] - 5s 85us/sample - loss: 0.3335 - acc: 0.8780
Epoch 4/5
60000/60000 [=======] - 6s 103us/sample - loss: 0.3134 - acc: 0.8844
Epoch 5/5
60000/60000 [=======] - 6s 94us/sample - loss: 0.2931 - acc: 0.8926
```
模型完成训练后，您会在最后一个周期结束时看到准确率值。类似如上所示的 0.8926。这表明您的神经网络对训练数据进行分类时的准确率约为 89%。换言之，它可以找出图像与标签之间的模式匹配，成功率达到 89%。

## 测试模型
调用model.evaluate 并传入测试集，并报告每组的损失。
```python
model.evaluate(test_images, test_labels)
```
输出内容如下所示：
10000/10000 [=====] - 1s 56us/sample - loss: 0.3365 - acc: 0.8789
[0.33648381242752073, 0.8789]

该示例返回了 0.8789 的准确率，表示其准确率约为 88%。


## 其他
- 经验法则1: 网络的第一层应与数据形状保持一致。图像的数据是28x28的图像，28 个包含 28 个神经元的层将不可行，因此将 28,28 平化为 784x1 更有意义。

- 经验法则2: 最后一层中的神经元数量应该与您要分类的类的数量相匹配。

- 经验法则3: In a nutshell, normalization reduces the complexity of the problem your network is trying to solve. This can potentially increase the accuracy of your model and speed up the training

