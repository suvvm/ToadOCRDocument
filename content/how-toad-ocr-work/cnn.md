---
title: "Cnn"
date: 2021-05-07T17:57:23+08:00
draft: false
---

# 卷积神经网络 Convolutional Neural Network

​	ToadOCREngine 使用两种神经网络完成手写体字符集识别，本章对卷积神经网络的实现进行介绍

## 一、卷积神经网络

​	对于一张手写体字符图像而言，需要检测图像中的特征来决定图像所属的字符类别，通常情况下这些特征都不是由整张图像来决定的，而是由一些局部的区域来决定的。这就是局部性

​	对于不同的图像，如果它们具有同样的特征，这些特征会出现在图像的不同位置，也就是说可以用同样的检测模式去检测不同图像的相同特征，只不过这些特征处于图像中的不同位置，但是特征检测所做的操作几乎一样。这就是相同性

​	对于一张大图像，如果我们进行下采样，那么图像的性质基本保持不变。这就是不变性。ToadOCR使用EMNIST数据集，对于28 * 28 的数据下采样方面的优点并未直观体现。

​	在处理图像时，让每一个神经元都与那个层中的所有神经元进行全连接是不现实。相反，让每个神经元只与输入数据的一个局部区域连接时可行的。其实这是因为图片特征的局部性，所以只需要通过局部就能提取出相应的特征。

​	与神经元连接的空间大小叫做神经元的感受野，它的大小是一个认为设置的超参数，这其实就是滤波器的宽和高。在深度方向上，大小总是和图片的深度一致。

​	池化层，为的是逐渐降低数据体的空间尺寸，这样就能够减少网络中的参数量，减少计算资源耗费，同时也能够有效地控制过拟合。ToadOCREngine采取的是3 * 3的小滤波器。

ToadOCREngine采取三层卷积(权重矩阵W0、W1、W2)一层全链接（权重矩阵W3）一层输出（权重矩阵W4）的结构，Out中保存每次的输出解码结果， OutVal为Out的数据张量指针，保证Out改变时上次预测结果不被gc回收。

### 1、神经网络设计

- 卷积神经网络结构设计代码

  ```go
  // CNN 四层卷积神经网络
  // 卷积层表达式DropOut(MaxPoll(\sigma(w*x)))
  // 退出与最大池化为同一层
  type CNN struct {
  	G	*gorgonia.ExprGraph				// 语法数学表达式图（可重用节点的AST）
  	W0, W1, W2, W3, W4 *gorgonia.Node	// 各层权重
  	D0, D1, D2, D3 float64				// 退出概率
  	Out *gorgonia.Node
  	OutVal gorgonia.Value
  	VM gorgonia.VM
  	TrainEpoch int
  	Lock sync.Mutex
  }
  ```

### 2、神经网络创建

​	神经网络创建主要实现对权重矩阵的初始化。对于前三层20%的激活归零，对于最后一层，希望55%的激活归零。这是卷积神经网络的惯用值，人为制造信息瓶颈，使得卷积圣经网络网络可以学习到真正重要的特征。

 	滤波器大小3 * 3 ，3 * 3是最通用的滤波器大小。

```go
// NewCNN 新建卷积神经网络
// w0 第0层权重(32, 1, 3, 3) 批次数（滤波器数量）32 颜色通道1（灰度图） 滤波器大小3 * 3
// 卷积层改变了输入形状下方第1层与第2层的形状要根据上一层输出进行修改
// w1 第1层权重(64, 32, 3, 3)
// w2 第2层权重(128, 64, 3, 3)
// 第2层将输出一个秩为4的张量，为了方便执行乘法在第3层将其重构为一个秩为2的张量
// w3 第3层权重(128 * 3 * 3, 625)
// 第4层进行输出解码得到结果概率
// w4 第4层权重(625, 62)
// d0～d3前三层20%的激活随机归零,最后一层55%的激活随机归零
//
// 入参
//	g *gorgonia.ExprGraph	// 语法数学表达式图(可重用节点的AST)
//
// 返回
//	*model.CNN				// 卷积神经网络
func NewCNN(g *gorgonia.ExprGraph) *model.CNN {
	// w0层权重
	w0 := gorgonia.NewTensor(g, tensor.Float64, 4,
		gorgonia.WithShape(32, 1, 3, 3), gorgonia.WithName("w0"),
		gorgonia.WithInit(gorgonia.GlorotN(1.0)))
	// w1层权重
	w1 := gorgonia.NewTensor(g, tensor.Float64, 4,
		gorgonia.WithShape(64, 32, 3, 3), gorgonia.WithName("w1"),
		gorgonia.WithInit(gorgonia.GlorotN(1.0)))
	w2 := gorgonia.NewTensor(g, tensor.Float64, 4,
		gorgonia.WithShape(128, 64, 3, 3), gorgonia.WithName("w2"),
		gorgonia.WithInit(gorgonia.GlorotN(1.0)))
	w3 := gorgonia.NewMatrix(g, tensor.Float64,
		gorgonia.WithShape(128 * 3 * 3, 625), gorgonia.WithName("w3"),
		gorgonia.WithInit(gorgonia.GlorotN(1.0)))
	w4 := gorgonia.NewMatrix(g, tensor.Float64,
		gorgonia.WithShape(625, common.EMNISTByClassNumLabels), gorgonia.WithName("w4"),
		gorgonia.WithInit(gorgonia.GlorotN(1.0)))
	return &model.CNN{
		G: g,
		W0: w0,
		W1: w1,
		W2: w2,
		W3: w3,
		W4: w4,
		D0: 0.2,
		D1: 0.2,
		D2: 0.2,
		D3: 0.55,
	}
}
```

### 