---
title: "OCR识别服务节点部署 Toad Ocr Engine Server"
date: 2021-05-08T16:55:13+08:00
draft: false
---

本章主要表述将Toad OCR Engine 图像识别添加为集群中的RPC服务的方法。

ToadOCR分布式集群使用ETCD进行搭建。所有操作的前提是完成本章目录下的环境搭建部分。

## 一、ToadOCREngine服务架构

![logical-structure](/images/toad_ocr_engine_logical_structure.png)

## 二、RPC Server 实现

### 1、服务注册