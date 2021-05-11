---
title: "图像预处理服务部署"
date: 2021-05-08T16:55:24+08:00
draft: false
---

# 图像预处理服务部署 Toad Ocr Preprocessor Server

本章主要表述将Toad OCR Processor 图像处理module添加为集群中的RPC服务的方法。

ToadOCR分布式集群使用ETCD进行搭建。所有操作的前提是完成本章目录下的环境搭建部分。

## 任务

识别图像中的文字区域，并输出28×28的二值图供ToadOCREngine使用

## 识别文字区域前的准备

### 一、二值化与形态学上的膨胀和腐蚀

- 二值化使得图像的像素值更单一、图像更简单。
    - 二值化有三种常用操作
	- 简单阈值，选取一个全局阈值,所有像素与这个阈值作比较，然后就把整幅图像分成了非黑即白的二值图像
	- 自适应阈值，通过规定一个区域大小，比较这个点与区域大小里面像素点的平均值（或者其他特征）的大小关系确定这个像素点是属于黑或者白（如果是二值情况）
	- Otsu’s二值化，此处不讨论
- 在黑底白字的情况下，膨胀即对前景做加粗处理，用于处理文字“缺陷”问题
- 在黑底白字的情况下，腐蚀即对前景做减细处理，用于处理文字“毛刺”或“噪点”问题
- 在白底黑字的情况下，腐蚀和膨胀对前景起到相反的效果
- 开运算和闭运算
    - 开运算：先腐蚀，再膨胀;闭运算：先膨胀，再腐蚀
    - 对于黑底白字的图像，开运算可以去掉无关的噪点;闭运算可以将文字缺陷的部分填充

### 二、图像处理

- 1、将图像转为灰度图
    ```go
    gocv.CvtColor(originalMat, &targetMat, gocv.ColorBGRToGray)
    ```
- 2、对图像进行二值化处理，将图像变为白底黑字
    - 这里使用自适应阈值
    ```go
    gocv.AdaptiveThreshold(originalMat, &targetMat, 225, gocv.AdaptiveThresholdGaussian, gocv.ThresholdBinary, 31, 16)
    ```
- 3、对图像进行两次闭运算，以去掉噪点
    ```go
    // 膨胀，去掉细节，前景是黑色的话，腐蚀操作会膨胀，膨胀操作会腐蚀
    dilation := gocv.NewMat()
    gocv.Dilate(targetMat, &dilation, gocv.GetStructuringElement(gocv.MorphRect, image.Point{4, 4}))
    // 腐蚀，加粗轮廓，如表格线等。
    erosion := gocv.NewMat()
    gocv.Erode(dilation, &erosion, gocv.GetStructuringElement(gocv.MorphRect, image.Point{4, 4}))
    // 再次膨胀，腐蚀细节
    dilation2 := gocv.NewMat()
    gocv.Dilate(erosion, &dilation2, gocv.GetStructuringElement(gocv.MorphRect, image.Point{4, 4}))
    // 腐蚀加粗轮廓
    erosion2 := gocv.NewMat()
    gocv.Erode(dilation2, &erosion2, gocv.GetStructuringElement(gocv.MorphRect, image.Point{5, 5}))
    ```
- 4、再次进行二值化处理，不过本次是将图像转变为黑底白字
    ```go
    gocv.AdaptiveThreshold(erosion2, &binary2, 225, gocv.AdaptiveThresholdGaussian, gocv.ThresholdBinaryInv, 31, 16)
    ```
- 5、对图像进行膨胀操作使文字轮廓凸显
    ```go
    dilation3 := gocv.NewMat()
    gocv.Dilate(binary2, &dilation3, gocv.GetStructuringElement(gocv.MorphRect, image.Point{12,12}))
    ```

### 三、文字区域识别

```go
/// 识别文字区域
///
/// 入参
///	img gocv.Mat // 待识别的图像，要求黑底白字
/// 返回
///	[]image.Rectangle // 文字矩形区域的数组
func findTextRegion(img gocv.Mat) []image.Rectangle {
    // 查找轮廓
    rects := make([]image.Rectangle, 0)
    contours := gocv.FindContours(img, gocv.RetrievalTree, gocv.ChainApproxSimple)
    contoursSlice := make([]gocv.PointVector, 0)
    for i := 0; i < contours.Size(); i++ {
	cnt := contours.At(i)

	contoursSlice = append(contoursSlice, cnt)
    }
    // 去掉重叠的识别不完整的区域
    rects = removeOverlappingRect(img, contoursSlice)
    return rects
}
/// 去掉重叠的识别不完整的区域
///
/// 入参
///	img gocv.Mat // 待识别的图像
///	rects []gocv.PointVector // 识别出来的多边形的数组
/// 返回
///	[]image.Rectangle // 去重的文字矩形区域的数组
func removeOverlappingRect(img gocv.Mat, rects []gocv.PointVector) []image.Rectangle {

    sort.Slice(rects, func(i, j int) bool {return gocv.ContourArea(rects[i]) > gocv.ContourArea(rects[j])})

    blankMat := gocv.Zeros(img.Rows(), img.Cols(), gocv.MatTypeCV8U)
    newRects := make([]image.Rectangle, 0)
    for i := 0; i < len(rects); i++ {
	polygon := gocv.MinAreaRect(rects[i])
	region := blankMat.Region(polygon.BoundingRect)

	if region.Mean().Val1 > 0 {
	    continue
	}

	gocv.Rectangle(&blankMat, polygon.BoundingRect, color.RGBA{0, 255, 255, 255}, int(gocv.Filled))
	newRects = append(newRects, polygon.BoundingRect)
    }
    return newRects
}
```

### 四、图像裁剪并压缩

- 利用步骤三得到的矩形区域数组(rects)对原图像(img)进行裁剪，得到文字区域单独的图像
- 对裁剪得到的图像进行二值化，转变为黑底白字的图像
- 将图像大小压缩为28×28
- 将图像转为byte数组
```go
imageSet := make([][]float64, 0)
// 裁剪
for _, rect := range rects {
    img_region := img.Region(rect)
    // 转为灰度图
    img_region = grayImage(img_region)
    // 二值化
    gocv.AdaptiveThreshold(img_region, &img_region, 225, gocv.AdaptiveThresholdGaussian, gocv.ThresholdBinaryInv, 31, 16)

    // 将图像压缩成28*28
    gocv.Resize(img_region, &img_region, image.Point{28, 28}, 0, 0, gocv.InterpolationLinear)

    imgFloat := make([]float64, 0)
    imgBytes := img_region.ToBytes()
    // 像素缩放
    for _, b := range imgBytes {
	imgFloat = append(imgFloat, PixelWeight(b))
    }
    if err != nil {
	log.Printf("fail to DataPtrFloat32 %v", err)
	return nil, err
    }

    log.Printf("append imageSet")
    imageSet = append(imageSet, imgFloat)
}
```
