---
title: "Install Dependencies"
date: 2021-05-04T16:35:56+08:00
draft: false
weight: 1
---

## 一、安装gRPC依赖

### 1、go

- 拥有Go三个最新发行版中的任何一个，有关安装说明请参考[Go Getting Start](https://golang.org/doc/install) 指导。

- 拥有可以使用的[protobuf](https://developers.google.com/protocol-buffers)编译器，protoc，[第3版](https://developers.google.com/protocol-buffers/docs/proto3)。有关安装说明，请参阅[protobuf编译器安装](https://grpc.io/docs/protoc-installation/)。

- 用于协议编译器的**Go插件**：

  1. 使用以下命令为Go安装proto编译器插件：

     ```shell
     export GO111MODULE=on  # Enable module mode
     go get google.golang.org/protobuf/cmd/protoc-gen-go \
              google.golang.org/grpc/cmd/protoc-gen-go-grpc
     ```

  2. 更新您的环境变量以便protoc编译器可以找到插件：

     ```shell
     export PATH="$PATH:$GOPATH/bin"
     ```

### 2、C++

​	C ++没有通用管理项目依赖项的标准。在构建和运行此快速入门的Ping Pong示例之前，您需要构建并安装gRPC。

​	构建和本地安装gRPC和协议缓冲区使用`cmake`。如果您愿意使用[bazel](https://www.bazel.build/)，请参阅[从源代码构建](https://github.com/grpc/grpc/blob/master/BUILDING.md#build-from-source)。

- 安装cmake

  - mac

    ```sh
    $ brew install cmake
    ```

  - linux

    ```shell
    $ sudo apt install -y cmake
    ```

  检查版本：

  ```sh
  $ cmake --version
  cmake version 3.19.6
  ```

- 安装其他工具

  - Linux

    ```sh
    $ sudo apt install -y build-essential autoconf libtool pkg-config
    ```

  - Mac：

    ```sh
    $ brew install autoconf automake libtool pkg-config
    ```

- 克隆grpc仓库

  ```sh
  $ git clone --recurse-submodules -b v1.35.0 https://github.com/grpc/grpc
  ```

- 构建并本地安装gRPC，protobuf和Abseil：

  ```sh
  $ cd grpc
  $ mkdir -p cmake/build
  $ pushd cmake/build
  $ cmake -DgRPC_INSTALL=ON \
        -DgRPC_BUILD_TESTS=OFF \
        -DCMAKE_INSTALL_PREFIX=$MY_INSTALL_DIR \
        ../..
  $ make -j
  $ make install
  $ popd
  $ mkdir -p third_party/abseil-cpp/cmake/build
  $ pushd third_party/abseil-cpp/cmake/build
  $ cmake -DCMAKE_INSTALL_PREFIX=$MY_INSTALL_DIR \
        -DCMAKE_POSITION_INDEPENDENT_CODE=TRUE \
        ../..
  $ make -j
  $ make install
  $ popd
  ```

### 3、Java

- 安装[JDK](https://jdk.java.net/) 7或更高版本

- maven

  ```xml
  <dependency>
      <groupId>io.grpc</groupId>
      <artifactId>grpc-all</artifactId>
      <version>1.26.0</version>
  </dependency>
  ```

- gradle

  ```
  protobuf {
      protoc {
          artifact = "com.google.protobuf:protoc:3.2.0"
      }
      plugins {
          grpc {
              artifact = 'io.grpc:protoc-gen-grpc-java:1.4.0'
          }
      }

      generatedFilesBaseDir = "src"

      generateProtoTasks {
          all()*.plugins {
              grpc {
                  outputSubDir = "java"
              }
          }
      }
  }
  ```

### 4、Python

- Python 3.5或更高版本

- pip 版本9.0.1或更高

- 安装gRPC：

  ```sh
  $ python -m pip install grpcio
  ```

- gRPC工具protobuf protoc

  ```sh
  $ python -m pip install grpcio-tools
  ```

## 二、Etcd Client v3

ToadOCR与Etcd的交互使用官方提供的etcd v3客户端包。

```sh
$ go get -u -v go.etcd.io/etcd/clientv3
```

​	但是当前etcd v3客户端的开发相对grpc略有滞后，且社区开发人员毫无干劲，积攒了巨量的issue没有处理，且对v3.5的发布日期来回扯皮详见https://github.com/etcd-io/etcd/issues/12330，所以当前clientv3对应相对稳定的gRPC版本为v1.26.0，若使用当前v1.3x.x会出现调用移除方法和generate代码变量名称不匹配的情况，需要手动对源码进行修改，得不偿失，所以可以考虑在go mod依赖管理中将gRPC版本限制为v1.26.0

```
replace (
	google.golang.org/grpc => google.golang.org/grpc v1.26.0
)
```

## 三、Gorgonia

​	ToadOCR为了方便进行数学运算，使用了gorgonia的张量与语法数学表达式图（可重用节点的AST）

```sh
$ go get gorgonia.org/gorgonia
```

## 四、GoCV

​	GoCV为[OpenCV 4](http://opencv.org/)计算机视觉库提供了Go语言绑定。GoCV软件包在Linux，macOS和Windows上支持Go和OpenCV v4.5.1的最新版本。在获取GoCV前请确保已经完成了OpenCV相关配置。

```sh
$ go get -u -d gocv.io/x/gocv
```

#### 快速安装OpenCV4

以下命令应执行所有操作，以在Linux上下载并安装OpenCV 4.5.2：

```sh
$ cd $GOPATH/src/gocv.io/x/gocv
$ make install
```

### 验证安装

```sh
$ go run ./cmd/version/main.go
```

输出

```
gocv version: 0.27.0
opencv lib version: 4.5.2
```

