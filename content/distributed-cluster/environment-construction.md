---
title: "服务器环境搭建"
date: 2021-05-08T16:53:51+08:00
draft: false
---

# 服务器环境搭建 Environment Construction

​	本章主要介绍搭建集群与部署节点前的环境搭建步骤

## 一、Install ECTD

### 1、Ubnutu\Debian

```sh
$ sudo apt-get install etcd
```

### 2、CentOS

```sh
$ yum install etcd
```

### 3、检查版本

```sh
$ etcd --version
```

输出

```
etcd Version: 3.2.26
Git SHA: your git version
Go Version: your go verson
Go OS/Arch: linux/amd64
```

## 二、Install Golang

### 1、Linux

- 下载go二进制包，您可以去[Go文档的安装页面](https://golang.org/doc/install)下载
```sh
# 您也可以使用wget
wget https://golang.org/dl/go1.15.11.linux-amd64.tar.gz
```
- 进入下载目录并在终端中输入如下代码安装Go的二进制包
```sh
# 您需要root权限来执行下面的代码
rm -rf /usr/local/go && tar -C /usr/local -xzf go1.16.4.linux-amd64.tar.gz
```
- 配置环境变量，让go命令可以随处被访问
```sh
# 在配置文件中加入如下代码，例如Ubuntu/Debian下的~/.bashrc
export PATH=$PATH:/usr/local/go/bin
```
- 您可以重启使环境变量生效，也可以使用`source`命令使之生效
### 2、检查版本

```sh
go version
```
输出
```sh
go version go1.15.11 linux/amd64
```


## 三、Install OpenCV and Gocv

### 安装Opencv(可选)

#### 1、源内安装

源内安装的版本通常较老，若有需求可以手动安装
- Ubnutu\Debian
```sh
sudo apt install libopecv-dev
```
- CentOS
```sh
sudo yum install opencv opencv-devel
```

#### 2、手动编译安装核心模块

- 安装依赖 cmake g++ wget unzip
    - Ubuntu/Debian:
    ```sh
    sudo apt update && sudo apt install -y cmake g++ wget unzip
    ```
    - Centos:
    ```sh
    # 注：Centos默认源内依赖版本可能较老，如有需要请更换源
    sudo yum check-update && sudo yum install gcc-c++ wget unzip
    ```

- 下载源码并解压
```sh
wget -O opencv.zip https://github.com/opencv/opencv/archive/master.zip
unzip opencv.zip
```
- 创建构建所需文件夹
```sh
mkdir -p build && cd build
```
- 配置
```sh
cmake  ../opencv-master
```
- 构建
```sh
cmake --build .
```
- 安装
```sh
sudo make install
```

更多请移步[opencv官方文档](https://docs.opencv.org/master/d7/d9f/tutorial_linux_install.html)
### 安装Gocv

```sh
go get -u -d gocv.io/x/gocv
```
如果您已安装好opencv,则可以跳过下面的内容；如果没有安装opencv,您可以编译安装gocv携带的opencv库
```sh
cd $GOPATH/src/gocv.io/x/gocv
make install
```
您也可以将opencv编译安装为静态依赖
```sh
make install BUILD_SHARED_LIBS=OFF
```
如果安装成功，在安装的最后会输出如下两行
```sh
gocv version: 0.27.0
opencv lib version: 4.5.2
```
