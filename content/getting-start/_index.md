---
title: "快速开始"
description: "ToadOCR分为两个部分，Preprocessor与OCREngine"
featured_image: ''
weight: 1
---
# 获取ToadOCR

​	ToadOCR暂时还没用添加对go-gettable的支持，若想要获取完整项目，则需要克隆github相关代码仓库。

- [ToadOCRProcessor](https://github.com/suvvm/ToadOCREngine)

  ```
  git clone git@github.com:suvvm/ToadOCREngine.git
  ```

- [ToadOCREngine]()

  ```
  git@github.com:suvvm/ToadOCREngine.git
  ```

## 使用客户端

​	若只使用RPC客户端，您也不需要在github克隆源码.

​	代码仓库中提供了Preprocessor和Engine的protobuf IDL，可以通过IDL生成gRPC客户端代码，详见教程[使用客户端]()

## 进行一次简单的调用
​	当前ToadOCR http服务部署在域名de.suvvm.work下

- 使用PostMan进行简单测试
![postman_process](/images/postman_process.png)
