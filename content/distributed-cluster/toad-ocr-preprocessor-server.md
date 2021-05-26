---
title: "图像预处理服务部署"
date: 2021-05-08T16:55:24+08:00
draft: false
---

# 图像预处理服务部署 Toad Ocr Preprocessor Server

本章主要表述将Toad OCR Processor 图像处理module添加为集群中的RPC服务的方法。

ToadOCR分布式集群使用ETCD进行搭建。所有操作的前提是完成本章目录下的环境搭建部分。

## 一、任务

识别图像中的文字区域，并输出28×28的二值图供ToadOCREngine使用

## 二、识别文字区域前的步骤

- 值化与形态学上的膨胀和腐蚀

- 图像处理（灰度化、二值化）

- 文字区域识别

- 图像裁剪并压缩   

## 三、RPC Server 实现

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