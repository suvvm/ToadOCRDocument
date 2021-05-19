---
title: "分布式集群"
date: 2021-05-08T15:46:21+08:00
draft: false
---

# Distributed cluster

​	本章包含的所有文章主要讲解ToadOCR分布式服务体系。

​	ToadOCR当前完成了分布式服务搭建与部署，单个节点与其他节点之间使用RPC进行交互。

### 文章目录

- [服务器环境搭建](environment-construction/)
- [中国大陆服务器代理](proxy-china)
- [集群搭建](cluster-construction/)
- [OCR识别服务节点部署](toad-ocr-engine-server/)
- [图像预处理服务节点部署](toad-ocr-preprocessor-server/)
- [缓存与持久化](cache-and-persistence/)

​	当前集群中共包含5台服务器（实际情况可能发生变动，ToadOCR开发组无法供给太多的服务器开销，若当前集群中的境外节点被GFW封禁，ToadOCR开发组会考虑将其关闭）

| 国家/地区 | 城市     | 供应商            | 系统              | 状态      |
| --------- | -------- | ----------------- | ----------------- | --------- |
| 中国      | 北京     | 阿里云轻量-北京   | Linux\CentOS7.2   | Online    |
| 中国香港  | 香港     | AWS亚太1区        | Linux\Debian10.2  | Unhealthy |
| 日本      | 东京     | Vultr东京         | Linux\Debian9.0   | Online    |
| 德国      | 法兰克福 | 腾讯云法兰克福1区 | Linux\Ubnutu20.04 | Online    |
| 美国      | 硅谷     | Vultr美西-硅谷    | Linux\Debian9.0   | Offline   |

### 分布式服务架构

![claster](/images/toad_ocr_call_chain.png)

### 分布式集群节点位置

![world-map](/images/worldMap.svg)

