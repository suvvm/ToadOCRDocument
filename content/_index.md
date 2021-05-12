---
title: "Taod OCR Document"
featured_image: '/images/gohugo-default-sample-hero-image.jpg'
description: "Go语言开发的神经网络OCR - 开放RPC接口供开发者使用 - 这是提供介绍和教程的网站"
---

​	欢迎来到Toad OCR的文档，署地址 http://vjp.suvvm.work/ ，Toad OCR的开发正在进行中。我一直在致力于提供完善的文档和接口描述。您可以阅读以下章节来了解ToadOCR

## Toad OCR概述

​	ToadOCR基于OCR设计领域的基本思路进行设计与编码，本项目意为对国内外关于深度学习发展历程和最新的研究成果进行整理和总结,以此学习与理解人工神经网络及经典的卷积神经网络所涉及到的概念和算法。

​	ToadOCR当前已经完成了对前馈型脉冲神经网络与卷积神经网络的支持。

 	ToadOCR在完成训练后，提供由全球多个国家5台服务器搭建分布式集群部署产物。负载均衡control center使用etcd，并开放gRPC接口

## 实现理论

研究方法：

- 预处理

  处理包含目标字符的图像，进行基本处理（降噪、灰度化、二值化、字符切分、归一化），最终将图像处理为规格相同的图像，以便后续得以顺利进行特征提取以及应用统一算法进行学习。

- OPENCV提取图像内待识别文字

  使用Canny算法进行多轮边缘查找，再将其查找结果（阈化图像）进行像素邻域计算以得到分布在图像内大大小小多个像素连通区域，然后将这些以矩形标识的分散区域根据识别对象字符横向等距分布的特性进行多次区域兼并运算，之后将多轮处理后得到的多个兼并对象作整体布局上的比较分析，从而淘汰不良结果。最后根据最优兼并结果内字符单元区域集合提取图像内待识别文字。

- 特征提取与降维

  提取图像中字符的特征作为识别字符的关键信息，用以标示每个独特的字符，若出现大字符集字符则进行特征降维以达到既保证分类器效率又保留了足够的信息量以区分文字。

- 设计分类器

  使用脉冲神经网络与卷积神经网络设计并训练分类器（监督学习）对提取的字符进行识别，来减轻步骤II特征工程的压力。

- 分类结果优化

  处理分类器得出的结果，通过语言模型对识别的字符进行矫正（尽力排除近形字），并将最终的识别结果进行格式化。

## 训练集合

 手写体训练数据分为两部分

- 数字数据使用使[MNIST](http://yann.lecun.com/exdb/mnist/)(Mixed National Institute of Standards and Technology database)数据集
- 字母与数字混合数据集使用[EMNIST](https://www.nist.gov/itl/products-and-services/emnist-dataset)(Extension Mixed National Institute of Standards and Technology database)数据集byClass部分。

