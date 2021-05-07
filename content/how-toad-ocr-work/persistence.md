---
title: "Persistence"
date: 2021-05-07T21:16:30+08:00
draft: false
---

# 神经网络权重持久化与恢复

​	各层的权重矩阵是神经网络训练结果的直观体现，可以通过对权重的复制得到一个完全相同的神经网络。本章将介绍ToadOCR针对支持的两种神经网络进行持久化和恢复的方式。

## 一、脉冲神经网络

### 1、持久化

​	ToadOCR前馈型脉冲神经设计为单隐层，只需将隐层Hidden与训练次数TrainEpoch进行保存即可，张量完成了对 `GobEncode` 的支持，可以直接使用`GobEncode`对张量进行持久化。

​	下方函数创建了文件"snn_weights"，并将隐层和训练次数持久化至其中

```go
func (snn *SNN) Persistence() error {
	if err := snnSave([]*tensor.Dense{snn.Hidden}, snn.TrainEpoch); err != nil {
		return err
	}
	return nil
}

func snnSave(nodes []*tensor.Dense, trainEpoch int) error {
	f, err := os.Create("snn_weights")
	if err != nil {
		return err
	}
	defer f.Close()
	enc := gob.NewEncoder(f)
	for _, node := range nodes {
		err := enc.Encode(node)
		if err != nil {
			return err
		}
	}
	err = enc.Encode(trainEpoch)
	if err != nil {
		return err
	}
	return nil
}
```

### 2、恢复

​	同理，张量完成了对 `GobDecode` 的支持，可以直接使用`GobDecode`对持久化的张量进行恢复。

​	下方函数读取snn_weights中持久化的权重矩阵与训练数据，并将其恢复为*model.SNN

```go
func LoadSNNFromSave() (*SNN, error) {
	f, err := os.Open("snn_weights")
	if err != nil {
		return nil, err
	}
	defer f.Close()
	dec := gob.NewDecoder(f)
	var hidden, final *tensor.Dense
	var trainEpoch int
	log.Println("decoding hidden")
	if err = dec.Decode(&hidden); err != nil {
		return nil, err
	}

	log.Println("decoding trainEpoch")
	if err = dec.Decode(&trainEpoch); err != nil {
		return nil, err
	}
	return &SNN{
		Hidden: hidden,
		Final: final,
		TrainEpoch: trainEpoch,
	}, nil
}
```

## 二、卷积神经网络

### 1、持久化

​	与脉冲神经网络相同，卷积神经网络的持久化也是真的权重矩阵与训练次数，这里将三个卷积层权重全链接层权重与输出权重全部进行持久化

```go
func (cnn *CNN) Persistence() error {
	if err := cnnSave([]*gorgonia.Node{cnn.W0, cnn.W1, cnn.W2, cnn.W3, cnn.W4}, cnn.TrainEpoch); err != nil {
		return err
	}
	return nil
}
```

### 2、恢复

​	与脉冲神经网络不通的是卷积神经网络在恢复时还需要恢复语法数学表达式图(可重用节点的AST)，并进行一次前向传播拿到OutVal的地址。并构造vm

```go
func LoadCNNFromSave() (*CNN, error) {
	f, err := os.Open("cnn_weights")
	if err != nil {
		return nil, err
	}
	defer f.Close()
	dec := gob.NewDecoder(f)
	var w0_val, w1_val, w2_val, w3_val, w4_val *tensor.Dense
	var trainEpoch int
	log.Println("decoding w0")
	if err = dec.Decode(&w0_val); err != nil {
		return nil, err
	}
	log.Println("decoding w1")
	if err = dec.Decode(&w1_val); err != nil {
		return nil, err
	}
	log.Println("decoding w2")
	if err = dec.Decode(&w2_val); err != nil {
		return nil, err
	}
	log.Println("decoding w3")
	if err = dec.Decode(&w3_val); err != nil {
		return nil, err
	}
	log.Println("decoding w4")
	if err = dec.Decode(&w4_val); err != nil {
		return nil, err
	}
	log.Println("decoding trainEpoch")
	if err = dec.Decode(&trainEpoch); err != nil {
		return nil, err
	}
	g := gorgonia.NewGraph()
	x := gorgonia.NewTensor(g, tensor.Float64, 4, gorgonia.WithShape(common.CNNBatchSize,
		common.RawImageChannel, common.RawImageRows,
		common.RawImageCols), gorgonia.WithName("x"))
	// 表达式网络输入数据y，内容为上述图像数据对应标签张量
	y := gorgonia.NewMatrix(g, tensor.Float64, gorgonia.WithShape(common.CNNBatchSize,
		common.EMNISTByClassNumLabels), gorgonia.WithName("y"))
	w0:= gorgonia.NewTensor(g, tensor.Float64,4, gorgonia.WithValue(w0_val), gorgonia.WithName("w0"))
	log.Printf(" w0:%v \n", w0)
	w1 := gorgonia.NewTensor(g, tensor.Float64,4, gorgonia.WithValue(w1_val), gorgonia.WithName("w1"))
	log.Printf("w1:%v \n", w1)
	w2 := gorgonia.NewTensor(g, tensor.Float64, 4, gorgonia.WithValue(w2_val), gorgonia.WithName("w2"))
	log.Printf("w2:%v \n", w2)
	w3 := gorgonia.NewMatrix(g, tensor.Float64, gorgonia.WithValue(w3_val), gorgonia.WithName("w3"))
	log.Printf("w3:%v \n", w3)
	w4 := gorgonia.NewMatrix(g, tensor.Float64, gorgonia.WithValue(w4_val), gorgonia.WithName("w4"))
	log.Printf("w4:%v \n", w4)
	cnn := &CNN{
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
		TrainEpoch: trainEpoch,
	}
	if err := cnn.Fwd(x); err != nil {
		return nil, err
	}
	losses := gorgonia.Must(gorgonia.HadamardProd(cnn.Out, y))
	cost := gorgonia.Must(gorgonia.Mean(losses))
	cost = gorgonia.Must(gorgonia.Neg(cost))
	// 跟踪成本
	var costVal gorgonia.Value
	gorgonia.Read(cost, &costVal)
	// 根据权重矩阵获取成本
	if _, err := gorgonia.Grad(cost, cnn.Learnables()...); err != nil {
		return nil, err
	}
	cnn.VMBuild()
	return cnn, nil
}
```

