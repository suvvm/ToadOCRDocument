---
title: "Toad Ocr 如何运作"
date: 2021-05-08T15:26:03+08:00
draft: false
---

# ToadOCR 如何运作

​	本章包含的所有文章主要讲解ToadOCR实现手写体预测的工作原理以及实现。

​	ToadOCR是一个有两种预测方式的OCR服务，其底层逻辑是创建两个神经网络（多层前馈脉冲神经网络、卷积神经网络），并使用EMNIST数据集训练神经网络而成。整体实现了数据集获取、数据集load、数据降维、神经网络结构设计与创建、前馈梯度下降训练、前馈预测、以及权重矩阵的持久化。

- [数据集load](data-load/)
- [图像可视化](visualize/)
- [图像的ZCA白化](zca/)
- [OpenCV图像处理](opencv_imgaes_processor/)
- [权重持久化与恢复](persistence/)
- [前馈型脉冲神经网络](snn/)
- [卷积神经网络](cnn/)