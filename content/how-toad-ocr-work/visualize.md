---
title: "Visualize"
date: 2021-05-07T21:09:36+08:00
draft: false
---

# 图像可视化

​	处理EMNIST数据集图像数据时，可视化也非常有用，ToadOCREngine提供了图像可视化功能。

## 一、像素放大

​	之前读取图像数据为张量时使用了PiexlWeight函数将图像像素从字节转换为float64类型并将范围缩放至0.0~1.0，若想进行图像可视化，则需要翻转函数

```go
// ReversePixelWeight 像素灰度反向放大函数
// 将缩放后的像素灰度浮点恢复为8位字节灰度值
//
// 入参
//	px float64	// 浮点灰度值
//
// 返回
//	byte		// 8位字节灰度值
func ReversePixelWeight(px float64) byte {
	return byte((px - 0.001) / 0.999 * common.PixelRange)
}
```

## 二、实现可视化

​	EMNIST数据集合在load后是一个巨大的图像切片，需要先计算出要可视化的数据（每行图像数量和每列图像数量）。在获得所需要的数据之后，通过go对切片的slice将张量进行分割，之后重新构造为一个（rows,cols,28,28）的4维张量。

​	之后可以利用，go内置图像处理包创建灰度图像 `canvas := image.NewGray(rect)`。image.Gray其实是一个字节型的切片，且每个字节是一个像素。之后循环遍历每块中的行和列，从张量职工提取正确的值进行填充，并通过上方ReversePixelWeight函数将浮点反向放大并转换为字节，之后将字节转换为color.Gray。最后利用canvas.Set(x, y, c)设置画布中的像素即可完成可视化编码。

```go
// Visualize 图像可视化实现函数
// 给定行数和列数，在float64张量data中获取图像，对像素灰度反向放大，
// 最终拼接为一个包含rows * cols个原始图像，文件名为filename的png图像
//
// 入参
//	data tensor.Tensor	// 保存所有图像float64数据的张量
//	rows int			// 目标图像每行所包含原始图像的数量
//	cols int			// 目标图像每列所包含原始图像的数量
//	filename string		// 目标图像文件名
//
// 返回
// error				// 错误信息
func Visualize(data tensor.Tensor, rows, cols int, filename string) error {
	imageTotal := rows * cols	// 计算图像包含原始图像总数
	sliced := data
	var err error
	if imageTotal > 1 {
		sliced, err = data.Slice(model.MakeRS(0, imageTotal), nil)	// 对data张量切片
		if err != nil {
			return err
		}
	}
	// 将分割后的张量重新构造为一个四维切片，可以认为这是一个rows * cols的，
	// 由rawImageRows * rawImageCols的原始图像组成的行和列
	if err = sliced.Reshape(rows, cols, common.RawImageRows, common.RawImageCols); err != nil {
		return err
	}
	imRows := common.RawImageRows * rows
	imCols := common.RawImageCols * cols
	rect := image.Rect(0, 0, imCols, imRows)	// 构造对应大小的矩形图像
	canvas := image.NewGray(rect)	// 根据构造的矩形创建灰度图画布
	// 遍历图像才张量中提取像素灰度值进行填充
	for i := 0; i < cols; i++ {
		for j := 0; j < rows; j++ {
			var patch tensor.Tensor
			if patch, err = sliced.Slice(model.MakeRS(i, i + 1), model.MakeRS(j, j + 1)); err != nil {
				return err
			}
			patchData := patch.Data().([]float64)
			for k, px := range patchData {
				x := j * common.RawImageRows + k % common.RawImageRows
				y := i * common.RawImageCols + k / common.RawImageCols
				c := color.Gray{ReversePixelWeight(px)}
				canvas.Set(x, y, c)
			}
		}
	}
	// 构造png图像文件
	var file io.WriteCloser
	if file, err = os.Create("output/images/"+filename); err != nil {
		return err
	}
	if err = png.Encode(file, canvas); err != nil {
		return err
	}
	if err = file.Close(); err != nil {
		return err
	}
	return nil
}
```

