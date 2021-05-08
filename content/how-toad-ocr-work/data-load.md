---
title: "数据集 Data load"
date: 2021-05-07T20:24:58+08:00
draft: false
---


​	ToadOCREngine 使用EMNIST与MNIST两种数据集进行训练，本章将对数据加载部分进行介绍

## 一、数据集准备

​	ToadOCREngine使用了MNIST与EMNIST数据集的byclass部分，数据集文件已上传至ToadOCR服务器集群leader——北京服务器中，clone ToadOCREngine源码后使用makebile进行构建时会触发对应脚本，主动从集群中获取数据集。

​	用户也可以手动下载数据集并提取到resources对应目录中

- EMNIST
  - [ToadOCR集群下载](https://www.suvvm.work/sundry/emnist-byclass.zip)
  - [官网下载](https://www.nist.gov/itl/products-and-services/emnist-dataset)
- MNIST
  - [ToadOCR集群下载](https://www.suvvm.work/sundry/mnist.zip)
  - [官网下载](http://yann.lecun.com/exdb/mnist/)

（⚠️警告：数据集文件当前仅存在于集群leader——北京服务器中并未同步至香港、东京以及欧洲法兰克福的服务器中，且为了防止对服务器恶意攻击，限制了下载速度不超过1mb/s）

## 二、数据集加载

### 1、图像加载

- 读取文件

  ToadOCREngine会首先将图像文件读取为二进制字节数组（大端序），读取文件是会根基文件幻数判读取文件的正确性，并加载图像相关元数据（图像数量、图像长度、图像宽度）

```go
// ReadImageFile 读取图像文件
//
// 入参
//	reader io.Reader	// 可读出字节流的数据源
// 文件幻数、图像数量、图像长度、图像宽度都是文件中的元数据
//
// 返回
//	[]RawImage		// 读取的图像
//	error				// 错误信息
func ReadImageFile(reader io.Reader) ([]common.RawImage, error) {
	var magic, n, nrow, ncol int32
	// 在数据源reader中大端序读出文件幻数（长度为size of int32的数据）
	if err := binary.Read(reader, binary.BigEndian, &magic); err != nil {
		return nil, err
	}
	if magic != common.ImageMagic { // 文件幻数与图像文件幻数不符，证明当前读取的文件非图像文件
		return nil, os.ErrInvalid
	}
	// 在数据源reader中大端序读出图像数量
	if err := binary.Read(reader, binary.BigEndian, &n); err != nil {
		return nil, err
	}
	// 在数据源reader中大端序读出图像宽度
	if err := binary.Read(reader, binary.BigEndian, &nrow); err != nil {
		return nil, err
	}
	// 在数据源reader中大端序读出图像高度
	if err := binary.Read(reader, binary.BigEndian, &ncol); err != nil {
		return nil, err
	}
	images := make([]common.RawImage, n)
	imageSize := int(nrow * ncol)	// 图像大小
	for i := 0; i < int(n); i++ {
		images[i] = make(common.RawImage, imageSize)
		readSize, err := io.ReadFull(reader, images[i])	// 读取一个图像
		if err != nil {
			return nil, err
		}
		if readSize != imageSize {	// 读取长度与图像大小不一致
			return nil, os.ErrInvalid
		}
	}
	return images, nil
}
```

- 构造张量

  ​	完成数据读取后会将得到的字节数组进行缩放得到\[]float64支撑数组切片，之后构造为方便运算的张量。

  ​	技术上来说，只需要将图像数据矩阵记为\[]\[]float64即可用于神经网络运算。但经过大约40年时间来开发的高效线性代数运算算法（BLAS）已经非常成熟，对矩阵向量乘法，ToadOCR使用 gorgonia的张量。张量类似于向量，为了方便理解这里可以吧张量理解为矩阵，具体张量设计格式相见 [gorgonia.tensor](https://github.com/gorgonia/tensor#design-of-dense)。

  （ps.为何不使用gonum的mat？因为gonum的mat多用于2维张量的运算，而ToadOCR神经网络运算中多为4维张量运算）

```go
// PrepareX 将图像数组转化为tensor存储的矩阵
// tensor设计格式详见 https://github.com/gorgonia/tensor#design-of-dense
//
// 入参
//	images []RawImage	// 完成初始化的mnist图像数组
//
// 返回
//	tensor.Tensor	// tensor矩阵
func PrepareX(images []common.RawImage) tensor.Tensor {
	rows := len(images)	// 矩阵宽度
	cols := len(images[0])	// 矩阵高度
	// 创建矩阵的支撑平面切片，tensor会复用当前切片的空间
	supportSlice := make([]float64, 0, rows * cols)
	for i := 0; i < rows; i++ {	// 复制缩放后的像素切片进入矩阵平面切片
		for j := 0; j < len(images[i]); j++ {
			supportSlice = append(supportSlice, PixelWeight(images[i][j]))
		}
	}
	// 返回rows * cols的tensor矩阵
	return tensor.New(tensor.WithShape(rows, cols), tensor.WithBacking(supportSlice))
}
```

​	图像缩放算法主要讲图像像素值范围由0～255缩小至0.0~1.0，且另外提供了一种额外的检查，如果要返回的值为1.0将返回0.999，而不是1。这主要是因为当值为1.0是数学函数性能表现可能不正常。

### 2、标签加载

- 读取文件

​	ToadOCREngine会首先将标签文件读取为二进制字节数组（大端序），读取文件是会根基文件幻数判读取文件的正确性，并加载标签相关元数据（标签数量）

```go
// ReadLabelFile 读取标签文件
//
// 入参
//	reader io.Reader	// 可读出字节流的数据源
// 文件幻数、标签数量都是文件中的元数据
//
// 返回
//	[]Label		// 读取的标签
//	error				// 错误信息
func ReadLabelFile(reader io.Reader) ([]common.Label, error) {
	var magic, n int32
	// 在数据源reader中大端序读出文件幻数（长度为size of int32的数据）
	if err := binary.Read(reader, binary.BigEndian, &magic); err != nil {
		return nil, err
	}
	if magic != common.LabelMagic { // 文件幻数与标签文件幻数不符，证明当前读取的文件非标签文件
		return nil, os.ErrInvalid
	}
	// 在数据源reader中大端序读出标签数量
	if err :=  binary.Read(reader, binary.BigEndian, &n); err != nil {
		return nil, err
	}
	labels := make([]common.Label, n)
	// 读取标签至labels
	for i := 0; i < int(n); i++ {
		var l common.Label
		if err := binary.Read(reader, binary.BigEndian, &l); err != nil {
			return nil, err
		}
		labels[i] = l
	}
	return labels, nil
}
```

- 构造张量

  ​	完成数据读取后会将得到的字节数组进行缩放得到\[]float64支撑数组切片，并将标签记为热独向量，之后构造为方便运算的张量。

  ​	为了方便运算，读取标签时会对标签进行热独编码得到热独向量，假设一个标签是9，那么它的热独向量编码就是\[0,0,0,0,0,0,0,0,0,1]。

  