---
title: ML入门-使用卷积和池化增强计算机视觉模型
date: 2023-12-05 19:39:12
tags:
- ml
categories:
- Machine Learning
---
## 什么是卷积

卷积是一种过滤条件，可遍历图片、对其进行处理，然后提取重要的特征。事实上，卷积就是一种数学运算，跟加减乘除没有本质的区别。虽然这种运算本身很复杂，但它非常有助于简化更复杂的表达式.

假设您有一幅穿着运动鞋的人的图像。如何检测图像中是否存在运动鞋？为了让您的程序“看到”运动鞋的图像，您必须提取重要的特征，并对不必要的功能进行模糊处理。使用卷积输出的是一幅修改后的图像，这在深度学习中称为特征映射(feature map)。

从理论上说，特征映射过程非常简单。您将扫描图像中的每个像素，然后查看其相邻像素。您需将这些像素的值乘以过滤条件中的等效权重。

例如：

![图像上的卷积](https://developers.google.com/static/codelabs/tensorflow-3-convolutions/img/f7b0ab29e09a51f.png?hl=zh-cn)

在这种情况下，系统会指定 3x3 的卷积矩阵（称之为卷积核）。卷积核的大小和模式为一个搅拌图片的方法。
> - 单通道图片俗称灰度图，图片由二维矩阵构成，每个像素点都有一个值表示颜色。  
> - 三通道图片：三通道分别指RGB(红，绿，蓝)通道。将通道红绿蓝三通道比作三个手电筒，那么RGB的值就是三个手电筒的灯光亮度。如果R,G,B三个通道的亮度一致，即R=G=B，那么这样的图片就是灰度模式的图片。如果这三个值不相等，那么就是彩色图。三通道图片由三个矩阵构成，其中每个元素都是0到255之间的一个整数。

当前的像素值为 192。您可以通过以下方式计算新像素的值：查看邻域值，将其乘以过滤条件中指定的值，并使新像素值设为最终值。(不同于矩阵乘法，却类似向量内积，这里是两个相同大小的矩阵的“点乘”)。当一个像素计算完毕后，移动一个像素到下一个区块执行相同的运算。

![](https://pic1.zhimg.com/v2-625af6a961d1476dfff000ee0d936ce8_b.webp)

通常卷积神经网络并不学习单一的核，而是同时学习多层级的多个核。比如一个32x16x16的核用到256×256的图像上去会产生32个241×241个feature map。
> feature map的大小计算公式为：image size - kernel size +1。

现在，您可以通过对 2D 灰度图像创建基本卷积来探索卷积的工作原理。

您将使用 [SciPy 的升序图像](https://docs.scipy.org/doc/scipy/reference/generated/scipy.misc.ascent.html)来进行演示。这是一个很好的内置图片工具，有许多角度和线条。

### 卷积的应用场景
1. 信号分析一个输入信号f(t)，经过一个线性系统（其特征可以用单位冲击响应函数g(t)描述）以后，输出信号应该是什么？实际上通过卷积运算就可以得到输出信号。
2. 图像处理输入一幅图像f(x,y)，经过特定设计的卷积核g(x,y)进行卷积处理以后，输出图像将会得到模糊，边缘强化等各种效果。

## 开始编码

首先导入一些 Python 库和升序图片：

```python
import cv2
import numpy as np
from scipy import misc
i = misc.ascent()
```

接下来，使用 Pyplot 库 `matplotlib` 绘制图像，以便您了解图像：

```python
import matplotlib.pyplot as plt
plt.grid(False)
plt.gray()
plt.axis('off')
plt.imshow(i)
plt.show()
```

![edb460dd5397f7f4.png](https://developers.google.com/static/codelabs/tensorflow-3-convolutions/img/edb460dd5397f7f4.png?hl=zh-cn)

您可以看到这是一张楼梯井的图像。您可以尝试并查明大量特征。例如，有很强的垂直线。

图像会存储为 NumPy 数组，因此我们只需复制该数组即可创建转换后的图像。size\_x 和 size\_y 变量将存储图像的尺寸，以便您稍后循环遍历该图像。

```python
i_transformed = np.copy(i)
size_x = i_transformed.shape[0]
size_y = i_transformed.shape[1]
```


## 创建卷积矩阵

首先，将卷积矩阵（或内核）构建成 3x3 数组：

```python
# This filter detects edges nicely
# It creates a filter that only passes through sharp edges and straight lines.
# Experiment with different values for fun effects.  
filter = [ [0, 1, 0], [1, -4, 1], [0, 1, 0]]
# A couple more filters to try for fun!filter = [ [-1, -2, -1], [0, 0, 0], [1, 2, 1]]
#filter = [ [-1, 0, 1], [-2, 0, 2], [-1, 0, 1]]
# If all the digits in the filter don't add up to 0 or 1, you
# should probably do a weight to get it to do so
# so, for example, if your weights are 1,1,1 1,2,1 1,1,1
# They add up to 10, so you would set a weight of .1 if you want to normalize them
weight  = 1
```

现在，计算输出像素。迭代图像，保留 1 像素的外边距，并将当前像素的每个邻域乘以该过滤条件中定义的值。

这意味着当前像素的邻近像素和位于左侧的像素将乘以过滤条件中的左上角项。然后，用所得结果乘以权重，确保结果介于 0 到 255 之间。

最后，将新值加载到转换后的图像中：

```python
    for x in range(1, size_x - 1):
        for y in range(1, size_y - 1):
            output_pixel = 0.0
            # 3x3 metric cal
            output_pixel = output_pixel + (i[x - 1, y - 1] * filter[0][0])
            output_pixel = output_pixel + (i[x, y - 1] * filter[0][1])
            output_pixel = output_pixel + (i[x + 1, y - 1] * filter[0][2])
            output_pixel = output_pixel + (i[x - 1, y] * filter[1][0])
            output_pixel = output_pixel + (i[x, y] * filter[1][1])
            output_pixel = output_pixel + (i[x + 1, y] * filter[1][2])
            output_pixel = output_pixel + (i[x - 1, y + 1] * filter[2][0])
            output_pixel = output_pixel + (i[x, y + 1] * filter[2][1])
            output_pixel = output_pixel + (i[x + 1, y + 1] * filter[2][2])
            output_pixel = output_pixel * weight
            if output_pixel < 0:
                output_pixel = 0
            if output_pixel > 255:
                output_pixel = 255
            i_transformed[x, y] = output_pixel
```

## [查看返回的结果](https://developers.google.com/codelabs/tensorflow-3-convolutions?hl=zh-cn#4)

现在，绘制图像，查看通过过滤条件的效果：

```python
# Plot the image. Note the size of the axes -- they are 512 by 512
plt.gray()
plt.grid(False)
plt.imshow(i_transformed)
#plt.axis('off')
plt.show()
```

![48ff667b2df812ad.png](https://developers.google.com/static/codelabs/tensorflow-3-convolutions/img/48ff667b2df812ad.png?hl=zh-cn)

请考虑以下过滤条件值及其对图像的影响。

使用 \[-1,0,1,-2,0,2,-1,0,1\] 获得一组很强的垂直线：

![垂直线检测过滤条件](https://developers.google.com/static/codelabs/tensorflow-3-convolutions/img/6f1cc59befac4d33.png?hl=zh-cn)

使用 \[-1,-2,-1,0,0,0,1,2,1\] 获得水平线：

![水平线检测](https://developers.google.com/static/codelabs/tensorflow-3-convolutions/img/705ec1004fce21d4.png?hl=zh-cn)

探索不同的值！此外，您还可以尝试其他尺寸的过滤条件，例如 5x5 或 7x7。

## [了解池化](https://developers.google.com/codelabs/tensorflow-3-convolutions?hl=zh-cn#5)

现在，您已确定了图像的基本特征，接下来要做什么？如何使用生成的特征图对图像进行分类？

与卷积类似，池化对特征检测有很大帮助。池化层可减少图像中的所有信息量，同时维持检测到的特征。

池化类型有多种，但您可以使用名为“最大池化”的池化类型。

迭代图片，并在每个点上考虑像素及其右侧、下方和右下方直接邻近像素。取其中最大的（即最大池化），并将其加载到新图像中。因此，新图像的大小将是旧图像的四分之一。![最大池化](https://developers.google.com/static/codelabs/tensorflow-3-convolutions/img/6029904d82700d8e.png?hl=zh-cn)

## [编写池化代码](https://developers.google.com/codelabs/tensorflow-3-convolutions?hl=zh-cn#6)

以下代码将显示 (2, 2) 池化。运行该代码以查看输出。

您会发现，虽然此图像大小是原始大小的四分之一，但保留了所有特征。

```python
    new_x = int(size_x / 2)
    new_y = int(size_y / 2)
    newImage = np.zeros((new_x, new_y))
    for x in range(0, size_x, 2):
        for y in range(0, size_y, 2):
            pixels = []
            pixels.append(i_transformed[x, y])
            pixels.append(i_transformed[x + 1, y])
            pixels.append(i_transformed[x, y + 1])
            pixels.append(i_transformed[x + 1, y + 1])
            pixels.sort(reverse=True)
            newImage[int(x / 2), int(y / 2)] = pixels[0]

    # Plot the image. Note the size of the axes -- now 256 pixels instead of 512
    plt.gray()
    plt.grid(False)
    plt.imshow(newImage)
    # plt.axis('off')
    plt.show()
```

![1f5ebdafd1db2595.png](https://developers.google.com/static/codelabs/tensorflow-3-convolutions/img/1f5ebdafd1db2595.png?hl=zh-cn)

请注意该图表的轴。图像现在的尺寸为 256x256，是原始尺寸的四分之一，尽管现在图像中显示的数据较少，但检测到的特征也得到了增强。

## 快速傅立叶变换
快速傅里叶变换是一种将时域和空域中的数据转换到频域上去的算法。傅里叶变换用一些正弦和余弦波的和来表示原函数。你可以在下面看到一个信号（一个以时间为参数的有周期的函数通常称为信号）是如何被傅里叶变换的：
![](https://pic4.zhimg.com/v2-45259486c87abae677e7f4f4bb89176b_b.webp)

红色是时域，蓝色为频域

## 三通道图片卷积操作
![](https://s2.51cto.com/images/blog/202302/15152631_63ec8927df6f311186.jpg?x-oss-process=image/watermark,size_16,text_QDUxQ1RP5Y2a5a6i,color_FFFFFF,t_30,g_se,x_10,y_10,shadow_20,type_ZmFuZ3poZW5naGVpdGk=/format,webp/resize,m_fixed,w_1184)

左列的X是输入的三通道图像即彩色图像，中间红色的两列是我们的kernel（即3x3的filter），共两个（即输出的feature通道为2）。最后一列为卷积之后的特征（由于2个kernel，输出通道为2）。
由上面的过程可以看出，输入是3维（hight*width*channel）的，kernel实际上也是三维的。