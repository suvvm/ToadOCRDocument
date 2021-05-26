---
title: "各服务描述"
date: 2021-05-20T14:15:37+08:00
draft: false
---

# ToadOCR各服务描述

本章主要介绍Toad OCR手写体识别服务体系各个独立服务，若有技术相关问题请在对应仓库提issue

## 一、ToadOCREngine

​	仓库地址 [https://github.com/suvvm/ToadOCREngine](https://github.com/suvvm/ToadOCREngine)

​	负责人: [@李亚宁](https://github.com/suvvm)

​	ToadOCREngine是集脉冲神经网络、卷积神经网络与RPC服务端为一体的手写体识别核心引擎，通过ToadOCREngine可以完成对神经网络的训练测试与服务端部署,具体描述详见仓库[Readme](https://github.com/suvvm/ToadOCREngine/blob/master/README.md).

- 构造命令(详见[makefile](https://github.com/suvvm/ToadOCREngine/blob/master/Makefile))

  - 关于构建二进制产物相关命令的帮助

    ```sh
    $ make help
    ```

  - 检查编译环境，编译并构建二进制产物

    ```sh
    $ make build
    ```

  - 编译并构建二进制产物后启动RPC服务端

    ```sh
    $ make run
    ```

  - 清除编译二进制产物与编译辅助产物

    ```sh
    $ make clean
    ```

  - 根据protobuf idl重新构造gRPC相关代码

    ```sh
    $ make generate
    ```

- 运行指令

  请在编译得到二进制产物./toad_ocr_engine后运行

  - ToadOCREngine帮助

    ```sh
    $ ./toad_ocr_engine help
    ```

  - 神经网络相关

    - 展示支持的神经网络列表

      ```sh
      $ ./toad_ocr_engine nnlist
      ```

    - 训练神经网络

      ```sh
      $ ./toad_ocr_engine train {{net_flag}}
      ```

    - 神经网络交叉验证

      ```sh
      $ ./toad_ocr_engine test {{net_flag}}
      ```

    - 清除持久化的权重数据

      ```sh
      $ ./toad_ocr_engine reset {{net_flag}}
      ```

      

  - RPC服务相关

    - 启动RPC服务端

      ```sh
      $ ./toad_ocr_engine server
      ```

    - 运行测试客户端

      ```sh
      $ ./toad_ocr_engine client
      ```

## 二、ToadOCRPreprocessor

​	仓库地址 [https://github.com/suvvm/ToadOCRPreprocessor](https://github.com/suvvm/ToadOCRPreprocessor)

​	负责人: [@郭一](https://github.com/beyond-1234)

​	ToadOCRPreprocessor 是集图像预处理与手写体识别调用RPC服务端RPC客户端为一体的OCR图像处理服务。其可以将拿到的图像进行腐蚀膨胀，并将文字区域分割为28*28的单个供下游ToadOCREngine使用，并构造最终识别结果。具体描述详见仓库[Readme](https://github.com/suvvm/ToadOCRPreprocessor/blob/master/README.md)

- 构造命令(详见[makefile](https://github.com/suvvm/ToadOCRPreprocessor/blob/master/Makefile))

  - 关于构建二进制产物相关命令的帮助

    ```sh
    $ make help
    ```

  - 检查编译环境，编译并构建二进制产物

    ```sh
    $ make build
    ```

  - 编译并构建二进制产物后启动RPC服务端

    ```sh
    $ make run
    ```

  - 清除编译二进制产物与编译辅助产物

    ```sh
    $ make clean
    ```

  - 根据protobuf idl重新构造gRPC相关代码

    ```sh
    $ make generate
    ```

- 运行指令

  请在编译得到二进制产物./toad_ocr_preprocessor后运行

  - ToadOCRPreprocessor帮助

    ```sh
    $ ./toad_ocr_preprocessor help
    ```

  - RPC服务相关

    - 启动RPC服务端

      ```sh
      $ ./toad_ocr_preprocessor server
      ```

    - 运行测试客户端

      ```sh
      $ ./toad_ocr_preprocessor client
      ```

## 三、ToadOCRRpcClient

​	仓库地址 [https://github.com/suvvm/ToadOCRRpcClient](https://github.com/suvvm/ToadOCRRpcClient)

​	负责人: [@李亚宁](https://github.com/suvvm)

​	ToadOCRRpcClient是ToadOCR PRC服务体系的客户端，其提供对ToadOCREngine与ToadOCRPreprocessor服务的调用方式，使得新服务可以快速接入ToadOCR PRC服务体系。

- 获取方式

  ```sh
   go get github.com/suvvm/ToadOCRRpcClient
  ```



## 四、ToadOCRTools

​	仓库地址 [https://github.com/suvvm/ToadOCRTools](https://github.com/suvvm/ToadOCRTools)

​	负责人: [@樊庆威](https://github.com/felixsfan)

​	ToadOCRTools是ToadOCR服务体系总开放的Http API，提供信息推送（邮件、短信）、身份识别信息管理、手写体识别调用相关接口，具体描述详见仓库[Readme](https://github.com/suvvm/ToadOCRTools/blob/master/README.md)

- 构造命令(详见[makefile](https://github.com/suvvm/ToadOCRTools/blob/master/Makefile))

  - 关于构建二进制产物相关命令的帮助

    ```sh
    $ make help
    ```

  - 检查编译环境，编译并构建二进制产物

    ```sh
    $ make build
    ```

  - 编译并构建二进制产物后启动RPC服务端

    ```sh
    $ make run
    ```

  - 清除编译二进制产物与编译辅助产物

    ```sh
    $ make clean
    ```

  - 根据protobuf idl重新构造gRPC相关代码

    ```sh
    $ make generate
    ```

- 运行指令

  请在编译得到二进制产物./toad_ocr_tools后运行

  - 启动http 服务端

    ```sh
    $ ./toad_ocr_tools
    ```

## 五、ToadOCRClientService

​	仓库地址 [https://github.com/felixsfan/ToadOCRClientService](https://github.com/felixsfan/ToadOCRClientService)

​	负责人: [@樊庆威](https://github.com/felixsfan)

​	部署站点地址: http://fqw.suvvm.work:8080/ocr_app/hello/

​    ToadOCRClientService是ToadOCR服务为了项目的演示和给用户提供免费的调用服务客户端。页面采用传统的HTML、CSS、Javascript和bootstrap响应式框架搭建，后台程序使用Python的Django（web框架）。具体描述详见仓库 [readme](https://github.com/felixsfan/ToadOCRClientService/blob/master/README.md)

- 构造和运行命令(详见[makefile](https://github.com/felixsfan/ToadOCRClientService/blob/master/Makefile.mk))

  - 运行项目

    ```sh
    $ make run
    ```

  - 清除清除日志和运行产生的文件

    ```sh
    $ make clean
    ```

## 六、ToadOCRDocument

​	仓库地址 [https://github.com/suvvm/ToadOCRDocument](https://github.com/suvvm/ToadOCRDocument)

​	负责人: [@郭一](https://github.com/beyond-1234)

​	部署站点地址: [http://vjp.suvvm.work/](http://vjp.suvvm.work/)

​	ToadOCRDocument是Toad OCR的文档，是用于提供介绍和教程的网站，网站由hugo支持，部署端提供webhook监听仓库的所有Push，master分支更新后会自动调用webhook拉去最新代码并部署。

