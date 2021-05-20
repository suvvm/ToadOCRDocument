---
title: "为何选择Golang"
date: 2021-05-19T16:05:42+08:00
draft: false
---

# 为何选择Golang Why Chose Golang

本章主要表述将Toad OCR手写体识别服务为何选择Golang作为开发语言。

## 一、机器学习

### 1、为何不使用Python

​	众所周知，Python是机器学习中最受欢迎的语言，但这并不意味着它是最好的。

​	Python成为使用广泛的机器学习语言很大程度上依赖于其数量庞大，种类繁多的库，这些库大多由C或C++编写，性能优异且可以完成几乎所有可以想到的训练任务。其次，Python是一门上手极度容易的语言，对没有使用过python甚至没有系统学习过编程却又有迫切开放需求的使用者（编码婴儿）来说非常友好。ToadOCR作者在某公司工作的过程中有过与Python http层老服务的对接工作，仅用了几小时，便可以独立完成对Python http层服务的开发与构建。

​	但是由于其定位问题Python的性能向来是遭到所有开发者诟病的一点，无论是臃肿的运行时还是愚蠢的伪并发，导致其除了IO密集型操作外性能方面效率极度低下。可以说Python在机器学习方向，脱离了各种库后便一无是处。

### 2、为何使用Golang

​	Python在机器学习中有极高的，但是人气与质量并不相同。尽管就流行度而言，Go[甚至没有](https://github.blog/2019-01-24-the-state-of-the-octoverse-machine-learning/)[跻身](https://websensa.com/en/2020/07/01/go-vs-python-which-one-is-better-for-machine-learning/)[于机器学习的十大语言](https://github.blog/2019-01-24-the-state-of-the-octoverse-machine-learning/)之列，但[Websensa在其帖子中承认](https://websensa.com/en/2020/07/01/go-vs-python-which-one-is-better-for-machine-learning/)， “Go更快，更具扩展性和高性能，因此它是大规模应用的理想选择”。

​	Go语言的相对来说是一种较为独特的编程，其在语法上类似于C，但是具有内存安全性，垃圾回收和结构化类型。除了极快的性能外，Go与Python不同由于goroutines和channel的存在，Go语言对并发的支持极度完备，单个go协程在启动后做占用的空间可以低至数kb，其对共享内存的多机器也有很好的支持。

​	虽然Go当前基本没有完善度极高的Machine Learning库，但是ToadOCR的宗旨是“独立设计数据结构与运算方式”，所以对于ToadOCR来说Go依旧是一种针对机器学习良好的编程语言。其语法简单，有助于清晰描述复杂的算法，同时又不至于使得开发人员无法理解如何优化代码与高效运行。

## 二、应用层面

​	Go语言是一种开源的通用语言，从脚本到Web到基架，几乎可以用它做任何事的。Go的设计它是为大型项目构建的。Go是一种过程性、功能性和并发性语言，而Python面向对象、功能性和命令性的语言。上方也提到过Python没有任何内置的并发机制。

​	且Go对不同平台的交叉编译有极好的支持，它允许使用静态链接，根据操作系统和体系结构的类型，将所有依赖项库和模块组合到一个单一的二进制文件中，使得Go产出的二进制产物完全没必要与Docker结合使用。（Docker是linux的著名容器，也是Go编写的）

​	ToadOCR在设计之初便不是提供一个单独的OCR应用，而是旨在部署一个较大型的，较完善的，多核心的分布式集群服务。而go语言在设计之初为的就是处理这种应用场景，go语言的设计旨在在[多核](https://en.wikipedia.org/wiki/Multi-core_processor)，[联网](https://en.wikipedia.org/wiki/Computer_network)[机器](https://en.wikipedia.org/wiki/Computer)和大型[代码库](https://en.wikipedia.org/wiki/Codebase)时代提高[编程效率](https://en.wikipedia.org/wiki/Programming_productivity)。使用go不会出现像c++一般“修改10分钟编译一整天”的奇特场景。

​	对于分布式服务来说，golang与当前使用最广泛的rpc服务体系grpc同属google服务体系，grpc对go语言有极好的支持。etcd站在zookeeper巨人的肩膀上。是一个高度一致的分布式键值存储，它由go编写提供了一种可靠的方式来存储需要由分布式系统或机器集群访问的数据。它可以优雅地处理网络分区（集群各个节点）之间的header选举，即使在leader节点中也可以容忍机器故障。（但是现在社区的开发者之间出现了慵懒的踢皮球行为，导致对grpc的新版本支持出现了些许问题，解决方案在“集群部署”部分）

