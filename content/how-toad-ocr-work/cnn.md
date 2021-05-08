---
title: "卷积神经网络 Convolutional Neural Network"
date: 2021-05-07T17:57:23+08:00
draft: false
---

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

### 3、前馈

​	前馈，就是数据在神经网络中传播的过程，如果这个过程中没有改变权重矩阵，则是一次预测过程。同理，如果前馈的过程中改变了权重矩阵的内容，则就是一次训练。

​	上方初始化每层权重的形状根据前馈函数计算得出，因为卷积层会改变输入的形状，ToadOCREngine使用EMNIST数据集，每个图像大小为28 * 28，给定第0层一个(N,1,28,28)的输入。32个3*3滤波器的卷积将会返回一个(N,32,14,14)的输出，经过形状为(2,2)的最大池化下采样后可以将图像长宽减半（总信息量减少1/4），而第1层与第2层与第0层很类似。

​	第2层输出是一个秩为4的张量，为了完成矩阵乘法进行输出解码，ToadOCREngine将其重新构造为一个秩为2的张量。

```go
// Fwd 前向传播函数
// 将图层包装在图层结构中执行每层激活
// 第0、1层，使用卷积神经网络的标准卷积stride =（1，1）和padding =（1，1）进行卷积
// 给定一个(N,1,28,28)的张量，卷积后得到一个(N,32,28,28)的张量
// 最大池化将图像长宽减半(2,2)后输出(M,32,14,14)
// 第3曾输出后将结果重构为一个秩为2的张量，使用softmax激活函数完成CNN实现
//
// 入参
//	x *gorgonia.Node	// 输入张量
//
// 返回
//	error				// 错误信息
func (cnn *CNN) Fwd (x *gorgonia.Node) error {
	// 第0层
	var c0, c1, c2, fc *gorgonia.Node
	var a0, a1, a2, a3 *gorgonia.Node
	var p0, p1, p2 *gorgonia.Node
	var l0, l1, l2, l3 *gorgonia.Node
	var err error

	log.Printf("x:%v", x)
	log.Printf("w0:%v", cnn.W0)

	if c0, err = gorgonia.Conv2d(x, cnn.W0, tensor.Shape{3, 3},
	[]int{1, 1}, []int{1, 1}, []int{1, 1}); err != nil {	// 卷积第0层
		return errors.Wrap(err, "Layer 0 Convolution failed")	// 跟踪错误信息
	}
	if a0, err = gorgonia.Rectify(c0); err != nil {	// 第0层激活
		return errors.Wrap(err, "Layer 0 activation failed")
	}
	if p0, err = gorgonia.MaxPool2D(a0, tensor.Shape{2, 2}, 	// 第0层最大池化
	[]int{0, 0}, []int{2, 2}); err != nil {
		return errors.Wrap(err, "Layer 0 Maxpooling failed")
	}
	if l0, err = gorgonia.Dropout(p0, cnn.D0); err != nil {	// 第0层概率退出
		return errors.Wrap(err, "Unable to apply a dropout to layer 0")
	}

	if c1, err = gorgonia.Conv2d(l0, cnn.W1, tensor.Shape{3, 3},	// 第1层卷积
	[]int{1, 1}, []int{1, 1}, []int{1, 1}); err != nil {
		return errors.Wrap(err, "Layer 1 Convolution failed")
	}
	if a1, err = gorgonia.Rectify(c1); err != nil {	// 第1层激活
		return errors.Wrap(err, "Layer 1 activation failed")
	}
	if p1, err = gorgonia.MaxPool2D(a1, tensor.Shape{2, 2}, 	// 第1层最大池化
	[]int{0, 0}, []int{2, 2}); err != nil {
		return errors.Wrap(err, "Layer 1 Maxpooling failed")
	}
	if l1, err = gorgonia.Dropout(p1, cnn.D1); err != nil {		// 第1层概率退出
		return errors.Wrap(err, "Unable to apply a dropout to layer 1")
	}


	if c2, err = gorgonia.Conv2d(l1, cnn.W2, tensor.Shape{3, 3}, 	// 第2层卷积
	[]int{1, 1}, []int{1, 1}, []int{1, 1}); err != nil {
		return errors.Wrap(err, "Layer 2 Convolution failed")
	}
	if a2, err = gorgonia.Rectify(c2); err != nil {	// 第2层激活
		return errors.Wrap(err, "Layer 2 activation failed")
	}
	if p2, err = gorgonia.MaxPool2D(a2, tensor.Shape{2, 2},	// 第2层最大池化
	[]int{0, 0}, []int{2, 2}); err != nil {
		return errors.Wrap(err, "Layer 2 Maxpooling failed")
	}
	log.Printf("p2 shape %v", p2.Shape())
	// 修改第2层张量结构，将输出的秩为4的张量重新构造为秩为2的蟑螂
	var r2 *gorgonia.Node
	b, c, h, w := p2.Shape()[0], p2.Shape()[1], p2.Shape()[2], p2.Shape()[3]
	if r2, err = gorgonia.Reshape(p2, tensor.Shape{b, c * h * w}); err != nil {
		return errors.Wrap(err, "Unable to reshape layer 2")
	}
	log.Printf("r2 shape %v", r2.Shape())
	if l2, err = gorgonia.Dropout(r2, cnn.D2); err != nil {	// 第2层概率退出
		return errors.Wrap(err, "Unable to apply a dropout on layer 2")
	}
	_ = ioutil.WriteFile("output/tmp.dot", []byte(cnn.G.ToDot()), 0644)
	// 第3层
	if fc, err = gorgonia.Mul(l2, cnn.W3); err != nil {
		return errors.Wrapf(err, "Unable to multiply l2 and w3")
	}
	if a3, err = gorgonia.Rectify(fc); err != nil {	// 第3层激活
		return errors.Wrapf(err, "Unable to activate fc")
	}
	if l3, err = gorgonia.Dropout(a3, cnn.D3); err != nil {	// 第3层概率退出
		return errors.Wrapf(err, "Unable to apply a dropout on layer 3")
	}
	// 输出解码
	var out *gorgonia.Node
	if out, err = gorgonia.Mul(l3, cnn.W4); err != nil {
		return errors.Wrapf(err, "Unable to multiply l3 and w4")
	}
	cnn.Out, err = gorgonia.SoftMax(out)	// 使用softmax 函数进行激活
	gorgonia.Read(cnn.Out, &cnn.OutVal)		// 保存预测结果
	return nil
}
```

### 4、训练步骤

使用命令

```sh
$ ./toad_ocr_engine train cnn
```

对卷积神经网络进行训练

1. 加载EMNIST数据集byclass图像与标签的训练集和测试集
   1. 加载图像
   2. 图像进行像素压缩算法
   3. 转换图像为张量
   4. 加载标签
   5. 转换标签为张量
2. 根据持久化的权重文件是否存在决定恢复网络或创建新的网络
3. 生成成本切片，使用本地迭代器对网络进行训练，训练过程中跟踪成本
4. 完成训练后对权重矩阵进行持久化
5. 使用测试集进行交叉验证

### 5、准确率

当前ToadOCREngine卷积神经网络训练使用EMNIST数据集byclass，训练Epoch200+ 准确率40%