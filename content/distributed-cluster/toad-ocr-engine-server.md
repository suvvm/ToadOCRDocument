---
title: "OCR识别服务节点部署"
date: 2021-05-08T16:55:13+08:00
draft: false
---

# OCR识别服务节点部署 Toad Ocr Engine Server

本章主要表述将Toad OCR Engine 图像识别添加为集群中的RPC服务的方法。

ToadOCR分布式集群使用ETCD进行搭建。所有操作的前提是完成本章目录下的环境搭建部分。

## 一、ToadOCREngine服务架构

![logical-structure](/images/toad_ocr_engine_logical_structure.png)

## 二、RPC Server 实现

### 1、服务注册与服务发现

​	详见本章下[Etcd gRPC 服务注册与服务发现](etcd-register-resolver/)

### 2、服务接口IDL

​	IDL中定义了SayHello与Predict两个接口

- SayHello

  是用于测试服务端是否运作正常的Ping Pong接口

  请求字段

  - name string：ping 信息

  响应字段

  - message string：pong信息

- Predict

  是用于预测单个图像的接口

  请求字段：

  - net_flag string： 预测使用网络标志，默认值为“snn”代表脉冲神经网络，当其值被置为“cnn”时，将使用卷积神经网络进行预测。
  - image []float64：图像数据，要求格式 28 * 28 黑底白字的灰度图，且图像像素以完成压缩（范围由0 ～ 255 压缩至 0.0～1.0）

```
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
option java_package = "work.suvvm.toad_ocr_engine";
option java_outer_classname = "ToadOcrProto";

package idl;

// The greeting service definition.
service ToadOcr {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
  rpc Predict (PredictRequest) returns (PredictReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}

message PredictRequest {
  string net_flag = 1;
  repeated double image = 2;
}

message PredictReply {
  int32 code = 1;
  string message = 2;
  string label = 3;
}

```

### 3、gRPC服务端代码生成

​	请确保开发机中拥有protobuf环境，且可以发现grpc的protobuf插件protoc-gen-go-grpc

- 生成go 服务端代码

```sh
$ protoc -I rpc/idl rpc/idl/toad_ocr.proto --go_out=plugins=grpc:rpc/idl
```

​	由于使用的grpc版本并非最新的release（详见环境搭建-Etcd client），所以要对生成的代码进行修正，将部分不正确的变量名进行替换

- 变量名修正

```sh
$ sed -i '' 's/ClientConnInterface/ClientConn/g' rpc/idl/toad_ocr.pb.go
$ sed -i '' 's/SupportPackageIsVersion6/SupportPackageIsVersion4/g' rpc/idl/toad_ocr.pb.go
```

### 4、handler设计

- SayHello

```go
// SayHello implements helloworld.GreeterServer
func (s *Server) SayHello(ctx context.Context, in *pb.HelloRequest) (*pb.HelloReply, error) {
	log.Printf("Received: %v", in.GetName())
	return &pb.HelloReply{Message: "Hello " + in.GetName()}, nil
}
```

- Predict

  Predict方法处理请求中的神经网络标识，根据标识分配给具体神经网络进行Predict，根据具体情况决定返回预测结果或错误信息。

```go
func (s *Server) Predict(ctx context.Context, in *pb.PredictRequest) (*pb.PredictReply, error) {
	log.Printf("Predict %v", in.NetFlag)
	var lab string
	var err error
	if in.NetFlag == common.SnnName {
		lab, err = nn.SnnPredict(snn, in.Image)
	} else if in.NetFlag == common.CnnName {
		cnn.Lock.Lock()
		lab, err = nn.CnnPredict(cnn, in.Image)
		cnn.Lock.Unlock()
	}
	if err != nil {
		return &pb.PredictReply{Code: int32(*errorCode), Message: err.Error(), Label: *errorLab}, nil
	}
	return &pb.PredictReply{Code: int32(*successCode), Message: *successMsg, Label: lab}, nil
}
```

### 5、服务端启动

- 初始化

  为了提高效率，ToadOCREngine服务端会长持两个神经网络，而不是每次收到请求后再尝试恢复它们

```go
func initNN() {
	var err error
	snn, err = model.LoadSNNFromSave()
	if err != nil {
		log.Fatalf("Failed at load snn weights %v", err)
	}
	cnn, err = model.LoadCNNFromSave()
	defer cnn.VM.Close()
	if err != nil {
		log.Fatalf("Unable to load cnn file %v", err)
	}
	var out gorgonia.Value
	rv := gorgonia.Read(cnn.Out, &out)
	fwdNet := cnn.G.SubgraphRoots(rv)
	vm := gorgonia.NewTapeMachine(fwdNet)
	cnn.VM = vm
}
```

- 启动逻辑
  - 进行初始化
  - 设置监听端口
  - 注册服务至控制中心
  - 启动新协程监听退出信号
  - 创建新的RPC server
  - 注册接口handler
  - 启动RPC服务端

```go
func RunRPCServer() {
	initNN()
	log.Printf("service listen port:%v", port)
	lis, err := net.Listen("tcp", ":" + *port)
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}
	log.Printf("register rpc server to control center...")
	err = Register(*reg, *serv, *host, *port, time.Second*10, 15)
	if err != nil {
		panic(err)
	}
	ch := make(chan os.Signal, 1)
	signal.Notify(ch, syscall.SIGTERM, syscall.SIGINT, syscall.SIGHUP, syscall.SIGQUIT)
	go func() {
		s := <-ch
		log.Printf("receive signal '%v'", s)
		UnRegister()
		os.Exit(1)
	}()
	log.Printf("create new toad ocr rpc server...")
	s := grpc.NewServer()
	log.Printf("register handler...")
	pb.RegisterToadOcrServer(s, &Server{})
	log.Printf("run toad ocr rpc server...")
	if err := s.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %v", err)
	}
}
```

