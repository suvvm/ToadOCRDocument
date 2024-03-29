---
title: "缓存与持久化"
date: 2021-05-18T21:25:19+08:00
draft: false
---

# 缓存与持久化 Cache and Persistence

本章主要表述将Toad OCR分布式集群对于数据缓存与持久化的实现方式。

## 一、缓存

### 1、ToadOCR缓存理念

​	在Toad OCR的设计中，缓存是一个高速数据存储层，其中存储了数据子集，且通常是短暂性存储，这样日后再次请求该数据时，速度要比访问数据的主存储位置快。通过缓存，您可以高效地重用之前检索或计算的数据。

​	Toad OCR部署在分布式计算环境中，借助专用缓存层，系统和对应的各个服务模块（API-Tools、Processor、Engine）应用程序可以使自己的生命周期与缓存相互独立地运行。

### 2、ToadOCR缓存如何运作

​	Toad OCR缓存中的数据通常存储在 RAM（随机存取存储器）快速存取硬件中，并在本机SDD（固态硬盘）中留档，方便在集群恢复时快速恢复缓存数据。缓存的主要目的是减少对底层速度较慢的存储层的访问需求，以此来提高数据检索性能。

​	只有在缓存中无法发现目标信息时，才会去数据存储中心的DB中获取数据。

### 3、ToadOCR缓存的发布与命中

- 读取缓存

​	ToadOCR在识别调用方身份时会通过对应的key尝试在缓存中获取目标数据，以构造身份识别令牌。如果提取的数据在缓存中不存在，则说明缓存未命中。

- 缓存发布

  ​	若未命中缓存，ToadOCR的各个服务模块会尝试与位于北京的数据存储中心的DB建立连接，并获取对应数据。在DB拿到数据后会构造对应K-V并Put到集群中。

  ​	集群中单个节点服务器在收到缓存Put后会将对应缓存信息广播至所有服务器节点，各节点之间数据强一致。

### 4、ToadOCR缓存的恢复

​	集群中的各个节点会将缓存的k-v存储在"{config-dir}/data/member"中，在集群重启后快速恢复缓存数据。

## 二、持久化

​	ToadOCR所有的数据都会在初次构造时持久化至位于数据存储中心节点的DB中