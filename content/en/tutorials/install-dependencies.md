---
title: "Install Dependencies"
date: 2021-05-04T16:35:56+08:00
draft: false
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
  python -m pip install grpcio-tools
  ```

## 二、安装Ectd

​	ToadOCR提供了修改过的ETCD安装脚本和启动脚本，可以在[项目仓库](https://github.com/suvvm/ToadOCREngine/blob/master/resources/script/etcd_install.sh)获取

