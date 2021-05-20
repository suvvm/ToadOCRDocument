---
title: "Taod OCR Document"
featured_image: '/images/gohugo-default-sample-hero-image.jpg'
description: "Go语言开发的神经网络OCR - 开放RPC接口供开发者使用 - 这是提供介绍和教程的网站"
---

​	欢迎来到Toad OCR的文档，Toad OCR的开发正在进行中。我一直在致力于提供完善的文档和接口描述。您可以阅读以下章节来了解ToadOCR

## Toad OCR概述

​	ToadOCR基于OCR设计领域的基本思路进行设计与编码，本项目意为对国内外关于深度学习发展历程和最新的研究成果进行整理和总结,以此学习与理解人工神经网络及经典的卷积神经网络所涉及到的概念和算法。

​	ToadOCR当前已经完成了对前馈型脉冲神经网络与卷积神经网络的支持。

 	ToadOCR在完成训练后，提供由全球多个国家5台服务器搭建分布式集群部署产物。负载均衡control center使用etcd，并开放gRPC接口

## 实现理论

研究方法：（具体详见[ToadOCR如何运作](http://vjp.suvvm.work/how-toad-ocr-work/)）

- OPENCV提取图像内待识别文字区域

  ​	使用Canny算法进行多轮边缘查找，再将其查找结果（阈化图像）进行像素邻域计算以得到分布在图像内大大小小多个像素连通区域，然后将这些以矩形标识的分散区域根据识别对象字符横向等距分布的特性进行多次区域兼并运算，之后将多轮处理后得到的多个兼并对象作整体布局上的比较分析，从而淘汰不良结果。最后根据最优兼并结果内字符单元区域集合提取图像内待识别文字。

  理论基于论文

  ```
  Satoshi Suzuki and others. Topological structural analysis of digitized binary images by border following. Computer Vision, Graphics, and Image Processing, 30(1):32–46, 1985.
  ```

- 预处理

  ​	处理包含目标字符的图像，进行基本处理（降噪、ZCA灰度化、像素缩放、二值化、归一化），最终将图像处理为规格相同的图像(28 * 28)，并作为张量读取至内存，以便后续得以顺利进行特征提取以及应用统一算法进行学习。

- 设计分类器

  ​	使用脉冲神经网络与卷积神经网络设计并训练分类器（监督学习）对提取的字符进行识别。

- 分类结果优化

  ​	处理分类器得出的结果，通过语言模型对识别的字符进行矫正（尽力排除近形字），并将最终的识别结果进行格式化。
  
- 训练权限开放

  ​	训练集数据在ToadOCR北京数据中心挂靠为静态资源，构建二进制产物时会一并获取至本机，最终开放训练命令与更新权重矩阵的逻辑，便于开发者构建属于自己的服务体系。

- 分布式集群部署

  ​	etcd作为强一致分布式存储系统控制服务注册服务发现与kv存储，使得全球各大洲多个服务节点统一为一个分布式集群进行管理。

- 服务模块分解

  ​	将整个OCR手写体识别服务设计为由多个服务模块组成的RPC服务调用链路，允许开发者可以通过规定的鉴权方式，在服务客户端宿主机不再集群中的情况下，连接至ToadOCR分布式集群调用进行服务发现并与目标模块服务端建立连接。

- 身份验证与对称加密鉴权

  ​	RPC分布式服务体系身份验证采取AppID - AppKey - AppSecret模式，由于当前开放服务原因，内部模块无需进行进一步鉴权，故为方便数据持久化与集群缓存，省略AppKey采取AppID- AppSecret模式进行鉴权。

- Http service

  http service提供用于信息推送（邮件、短信）控制身份信息、与调用RPC服务的接口，供给Client与不便于接入rpc服务的开发者使用。

- Client

  ​	Client用于非开发者用户直接调用OCR服务与开发者申请鉴权所需验证信息，Client使用独立的用户-时效token模式进行鉴权。

## 训练集合

 手写体训练数据分为两部分

- 数字数据使用使[MNIST](http://yann.lecun.com/exdb/mnist/)(Mixed National Institute of Standards and Technology database)数据集
- 字母与数字混合数据集使用[EMNIST](https://www.nist.gov/itl/products-and-services/emnist-dataset)(Extension Mixed National Institute of Standards and Technology database)数据集byClass部分。

## 服务限制与缺陷

- ToadOCR图像解码处理体系限制采用jpeg
- ToadOCR图像文字区域提取采用OpenCV且在过程有多次腐蚀与膨胀操作，故要求图像分辨率尽可能高且字符间距尽可能不要过小。
- 神经网络采用ByClass进行分类，故相似字符如 `i I l 1` 可能预测不够准确

## 潜在迭代方向

- 进一步训练提升神经网络准确度
- 减缓cnn(结构修改自LeNet-5)输出层下降速度添加缓冲层
- 扩充支持神经网络类型
- 取消输入张量与输出张量的硬编码，使得ToadOCR脱离OCR的限制成为通用型神经网络服务。

