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

## 二、RPC Server 实现

### 1、服务注册与服务发现

​	详见本章下[Etcd gRPC 服务注册与服务发现](etcd-register-resolver/)

### 2、服务接口IDL

IDL中定义了Ping与Process两个接口

- Ping

  是用于测试服务端是否运作正常的Ping Pong接口

  请求字段

  - app_id string：调用方AppID身份信息
  - basic_token string：调用方认证token
  - name string：ping 信息

  响应字段

  - message string：pong 信息

- Process

  是用于执行图像文字预测的接口

  请求字段：

  - app_id string：调用方AppID身份信息
  - basic_token string：调用方认证token
  - net_flag string： 预测使用网络标志，默认值为“snn”代表脉冲神经网络，当其值被置为“cnn”时，将使用卷积神经网络进行预测。
  - image []byte：图像数据使用jpeg编码

```protobuf
// Copyright 2015 gRPC authors.
//
// Licensed under the Apache License, Version 2.0 (the "License");
// you may not use this file except in compliance with the License.
// You may obtain a copy of the License at
//
//     http://www.apache.org/licenses/LICENSE-2.0
//
// Unless required by applicable law or agreed to in writing, software
// distributed under the License is distributed on an "AS IS" BASIS,
// WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
// See the License for the specific language governing permissions and
// limitations under the License.

syntax = "proto3";

option go_package = "/";
option java_multiple_files = true;
option java_package = "work.suvvm.toad_ocr_preprocessor";
option java_outer_classname = "ToadOcrPreprocessorProto";

package idl;

service ToadOcrPreprocessor {
  rpc Ping (PingRequest) returns (PongReply) {}
  rpc Process (ProcessRequest) returns (ProcessReply) {}
}

message PingRequest {
  string app_id = 1;
  string basic_token = 2;
  string name = 3;
}

message PongReply {
  string message = 1;
}

message ProcessRequest {
  string app_id = 1;
  string basic_token = 2;
  string net_flag = 3;
  bytes image = 4;
}

message ProcessReply {
  int32 code = 1;
  string message = 2;
  repeated string labels = 3;
}
```

### 3、gRPC代码生成

#### 1、服务端

请确保开发机中拥有protobuf环境，且可以发现grpc的protobuf插件protoc-gen-go-grpc

- 生成go 服务端代码

```sh
$ protoc -I rpc/idl rpc/idl/toad_ocr_preprocessor.proto --go_out=plugins=grpc:rpc/idl
```

​	由于使用的grpc版本并非最新的release（详见环境搭建-Etcd client），所以要对生成的代码进行修正，将部分不正确的变量名进行替换

- 变量名修正

```sh
$ sed -i '' 's/ClientConnInterface/ClientConn/g' rpc/idl/toad_ocr_preprocessor.pb.go
$ sed -i '' 's/SupportPackageIsVersion6/SupportPackageIsVersion4/g' rpc/idl/toad_ocr_preprocessor.pb.go
```

#### 2、Engine客户端

- 生成go 客户端代码

```sh
$ protoc -I client/toad_ocr_engine/idl client/toad_ocr_engine/idl/toad_ocr.proto --go_out=plugins=grpc:client/toad_ocr_engine/idl
```

- 变量名修正

```sh
$ sed -i '' 's/ClientConnInterface/ClientConn/g' client/toad_ocr_engine/idl/toad_ocr.pb.go
$ sed -i '' 's/SupportPackageIsVersion6/SupportPackageIsVersion4/g' client/toad_ocr_engine/idl/toad_ocr.pb.go
```

### 4、handler设计

- 身份验证

  handler在拿到请求后会对调用方的身份进行校验，过程如下

  1. 根据请求传入的appID，尝试获取调用方对应appSecret
     - 在集群缓存的k-v中存在目标appID的键
       - 获取对应k-v返回appSecret值
     - 在集群缓存的k-v中不存在目标appID的键
       - 在DB中根据appID获取appSecret
       - 将appID - appSecret 缓存至集群中
       - 返回appSecret值
  2. 根据appSecret与请求数据内容进行MD5，得到hash串
     - 将appSecret、请求中的参数数据组合成一串字符
     - 对组合后的内容进行MD5编码
  3. 验证MD5编码是否与basic_token一致
     - 一致：继续进行下一步业务逻辑处理
     - 不一致：构造拒绝响应并返回

- Ping 简单Ping Pong接口设计同Engine SayHello

- Process

  Process借口处理请求中的身份信息与图片信息

  - 验证身份
  - 调用上方图像处理方法，识别图像中的文字区域，并输出一组28×28的二值图切片
  - 并发请求ToadOCREngine 处理切片中的所有图像
  - 等待所有OCR处理完毕后构造处理响应并返回OCR识别结果

### 5、服务端启动

- 初始化

  服务端在启动时会读取配置文件，初始化集群的client与DB链接

- 启动逻辑

  - 进行初始化
  - 探测ToadOCREngine服务端存活状况
  - 设置监听端口
  - 注册服务至控制中心
  - 启动新协程监听退出信号
  - 创建新的RPC server
  - 注册接口handler
  - 启动RPC服务端