---
title: "Spiking Neural Network"
date: 2021-05-07T16:54:39+08:00
draft: false
---

# 脉冲神经网络 Spiking Neural Network

​	ToadOCREngine 使用两种神经网络完成手写体字符集识别，本章对脉冲神经网络的实现进行介绍

## 一. 多层前馈型脉冲神经网络

​	ToadOCREngine 使用脉冲神经网络使用多层前馈型脉冲神经网络

​	在ToadOCREngine多层前馈脉冲神经网络结构中，网络中的神经元是分层排列的，输入层各神经元的脉冲序列表示对具体图像输入数据的编码，并将其输入脉冲神经网络的下一层。最后一层为输出层，该层各神经元输出的脉冲序列构成网络的输出，即62种可识别数子和字符的热独向量，详见[tags mapping](http://localhost:1313/getting-start/tags-map/)。输入层和输出层之间有一个隐藏层。

​	ToadOCREngine多层前馈脉冲神经网络，与传统的前馈人工神经网络中两个神经元之间仅有一个突触连接不同，脉冲神经网络可采用多突触连接的网络结构，两个神经元之间可以有多个突触连接，每个突触具有不同的延时和可修改的连接权值。多突触的不同延时使得突触前神经元输入的脉冲能够在更长的时间范围对突触后神经元的脉冲发放产生影响。突触前神经元传递的多个脉冲再根据突触权值的大小产生不同的突触后电位。

### 1、神经网络设计

- 多层前馈型脉冲神经网络结构示意图

  ![snn-layout](/images/snn-layout.png)

- 多层前馈型脉冲神经网络结构设计代码

  ```go
  // SNN 脉冲神经网络结构
  // 输入层是common.MNISTRawImageRows * common.MNISTRawImageCols个float64型切片
  // 前向馈入(矩阵惩罚后执行激活函数)形成隐层
  // 隐层继续馈入形成输出层（MNIST为62个float64组成的向量，就是标签的热独编码）
  //
  // 参数
  //	Hidden *tensor.Dense	// 输入层与隐层之间的权重矩阵
  //	Final *tensor.Dense	// 隐层与输出层之间的权重矩阵
  //	B0 float64				// 隐层偏置
  //	B1 float64				// 输出层偏置
  type SNN struct {
  	Hidden *tensor.Dense
  	Final *tensor.Dense
  	B0 float64
  	B1 float64
  	TrainEpoch int
  }
  ```

### 2、神经网络创建

​	神经网络创建主要实现对权重矩阵的初始化，这里单独设计了一个函数`FillRandom`该函数在utils包下，得以在给定范围内均匀分布的填充平均值到权重矩阵中

```go
// NewSNN 构建新的脉冲神经网络
//
// 入参
//	input int	// 输入数据长度
//	hidden int	// 隐层神经元数量
//	output int	// 输出层神经元数量
//
// 返回
//	*model.SNN	// 基础神经网络
func NewSNN(input, hidden, output int) *model.SNN {
	hiddenData := make([]float64, hidden * input)	// 隐层权重矩阵数据切片
	finalData := make([]float64, hidden * output)	// 输出层权重矩阵数据切片
	// 随机填充
	utils.FillRandom(hiddenData, float64(len(hiddenData)))
	utils.FillRandom(finalData, float64(len(finalData)))
	// 创建张量
	hiddenT := tensor.New(tensor.WithShape(hidden, input), tensor.WithBacking(hiddenData))
	finalT := tensor.New(tensor.WithShape(output, hidden), tensor.WithBacking(finalData))
	// 返回新的基础神经网络
	return &model.SNN{
		Hidden: hiddenT,
		Final: finalT,
	}
}
```

### 3、前馈

​	前馈，就是数据在神经网络中传播的过程，如果这个过程中没有改变权重矩阵，则是一次预测过程。同理，如果前馈的过程中改变了权重矩阵的内容，则就是一次训练。

​	在ToadOCREngine中，成本采用的是误差的平方和，误差指的是目标数据label与预测值热独向量之差，为了避免成本出现负值，对误差取平方。成本的计算与反向传播密切相关。

​	已知成本是误差的平方和（y为label热独向量，pred为预测结果热独向量）
$$
cost=(y - pred)^2
$$
​	成本对预测求导，可得
$$
\frac {\delta cost}{\delta pred} = 2(y - pred)
$$
​	对sigmoid函数求导，可得
$$
\frac {d\sigma(x)}{dx} = \sigma(x)(1-\sigma(x))
$$
由此可以推导出成本函数关于权重矩阵的导数。

- 激活函数

```go
// Sigmoid （Logistic）神经网络激活函数
// 用于隐层神经元输出，将给定float64实数映射到(0,1)
// 定义公式 $ S(x) = \frac{1}{1 + e^{-x}} $
//
// 入参
//	x float64	// 给定float64实数
//
// 返回
//	float64		// 映射结果
func Sigmoid(x float64) float64 {
	return 1 / (1 + math.Exp(-1 * x))
}
```

- 预测函数

```go
// Predict 前向传播函数
//
// 入参
//	data tensor.Tensor	// 输入参数张量
//
// 返回
//	int					// 输出层最大值索引
//	error				// 错误信息
func (snn *SNN) Predict(data tensor.Tensor) (int, error) {
	if data.Dims() != 1 {
		return -1, errors.New("Expected a vector")
	}
	var m Maybe
	hidden := m.Do(func() (tensor.Tensor, error) {
		return snn.Hidden.MatVecMul(data)	// 矩阵向量乘法 Hidden × data
	})
	hiddenSigmoid := m.Sigmoid(hidden)	// 执行激活函数形成隐层权重矩阵
	final := m.Do(func() (tensor.Tensor, error) {	// 矩阵向量乘法 Final × hiddenSigmoid
		return snn.Final.MatVecMul(hiddenSigmoid)
	})
	pred := m.Sigmoid(final)	// 执行激活函数得到输出层权重矩阵
	if m.err != nil {
		return -1, m.err
	}
	return vecf64.Argmax(pred.Data().([]float64)), nil
}
```

- 训练函数

```go
// Train 基础神经网络单次训练函数
// 进行单次训练并得到本次训练的成本（误差平方和对预测结果求偏导数）
//
// 入参
//	x tensor.Tensor		// 图像张量
//	y tensor.Tensor		// 标签张量
//	learnRate float64	// 梯度更新值（学习速率）
// 返回
//	float64				// 本次成本
//	error				// 错误信息
func (snn *SNN) Train(x, y tensor.Tensor, learnRate float64) (float64, error) {
	// 预测
	var m Maybe
	// 调整图像张量为一维张量
	m.Do(func() (tensor.Tensor, error) {
		err := x.Reshape(x.Shape()[0], 1)
		return x, err
	})
	// 调整标签张量为一维张量
	m.Do(func() (tensor.Tensor, error) {
		err := y.Reshape(common.EMNISTByClassNumLabels, 1)
		return y, err
	})
	// 隐层权重矩阵乘输入数据
	hidden := m.Do(func() (tensor.Tensor, error) {
		return tensor.MatMul(snn.Hidden, x)
	})
	// 执行激活函数
	hiddenSigmoid :=  m.Sigmoid(hidden)

	//hiddenSigmoid := m.Do(func() (tensor.Tensor, error) {
	//	return hidden.Apply(utils.Sigmoid, tensor.UseUnsafe())
	//})

	// 隐层输出层权重矩阵乘隐层数据矩阵
	final := m.Do(func() (tensor.Tensor, error) {
		return tensor.MatMul(snn.Final, hiddenSigmoid)
	})
	pred := m.Sigmoid(final)
	//pred := m.Do(func() (tensor.Tensor, error) {
	//	return final.Apply(utils.Sigmoid, tensor.UseUnsafe())
	//})
	// 反向传播
	var cost float64
	// 计算输出层误差， 标签减去结果得到误差
	outputErrors := m.Do(func() (tensor.Tensor, error) {
		return tensor.Sub(y, pred)
	})
	cost = utils.Sum(outputErrors.Data().([]float64))
	// 计算隐层误差
	hidErrs := m.Do(func() (tensor.Tensor, error) {
		// Final 转置
		if err := snn.Final.T(); err != nil {
			return nil, err
		}
		defer snn.Final.UT()	// 函数结束后取消转置
		// Final 转置乘输出层误差
		return tensor.MatMul(snn.Final, outputErrors)
	})
	if m.err != nil {
		return 0, m.err
	}
	// 结果矩阵的导数
	dpred := m.Do(func() (tensor.Tensor, error) {
		return pred.Apply(utils.DSigmoid, tensor.UseUnsafe())
	})
	// 元素乘法计算误差对结果矩阵的导数
	m.Do(func() (tensor.Tensor, error) {
		return tensor.Mul(dpred, outputErrors, tensor.UseUnsafe())
	})
	// 误差对隐层输出层权重矩阵的导数
	dpred_dfinal := m.Do(func() (tensor.Tensor, error) {
		if err := hiddenSigmoid.T(); err != nil {
			return nil, err
		}
		defer hiddenSigmoid.UT()
		// 误差乘输入层数据的转置
		return tensor.MatMul(outputErrors, hiddenSigmoid)
	})
	// 输入矩阵的导数
	dHiddenSigmoid := m.Do(func() (tensor.Tensor, error) {
		return hiddenSigmoid.Apply(utils.DSigmoid)
	})
	// 误差乘输入矩阵的导数
	m.Do(func() (tensor.Tensor, error) {
		return tensor.Mul(hidErrs, dHiddenSigmoid, tensor.UseUnsafe())
	})
	// 调整误差乘输入矩阵的导数张量为一维张量
	m.Do(func() (tensor.Tensor, error) {
		err := hidErrs.Reshape(hidErrs.Shape()[0], 1)
		return hidErrs, err
	})
	// 误差对输入层隐层权重矩阵的导数
	dcost_dhidden := m.Do(func() (tensor.Tensor, error) {
		if err := x.T(); err != nil {
			return nil, err
		}
		defer x.UT()
		return tensor.MatMul(hidErrs, x)
	})
	// 使用上述求得的导数作为梯度，更新输入矩阵
	m.Do(func() (tensor.Tensor, error) {
		return tensor.Mul(dpred_dfinal, learnRate, tensor.UseUnsafe())
	})
	m.Do(func() (tensor.Tensor, error) {
		return tensor.Mul(dcost_dhidden, learnRate, tensor.UseUnsafe())
	})
	m.Do(func() (tensor.Tensor, error) {
		return tensor.Add(snn.Final, dpred_dfinal, tensor.UseUnsafe())
	})
	m.Do(func() (tensor.Tensor, error) {
		return tensor.Add(snn.Hidden, dcost_dhidden, tensor.UseUnsafe())
	})
	return cost, m.err
}
```

可以看到其实训练就是多次predict后对权重矩阵进行梯度更新。

### 4、训练步骤

使用命令

```sh
$ ./toad_ocr_engine train snn
```

对脉冲神经网络进行训练

1. 加载EMNIST数据集byclass图像与标签的训练集和测试集
   1. 加载图像
   2. 图像进行像素压缩算法
   3. 转换图像为张量
   4. 加载标签
   5. 转换标签为张量
2. 对训练集图像进行ZCA白化处理
3. 根据持久化的权重文件是否存在决定恢复网络或创建新的网络
4. 生成成本切片，使用本地迭代器对网络进行训练，训练过程中跟踪成本
5. 完成训练后对权重矩阵进行持久化
6. 使用测试集进行交叉验证

### 5、准确率

当前ToadOCREngine脉冲神经网络训练使用EMNIST数据集byclass，训练Epoch100+ 准确率82%