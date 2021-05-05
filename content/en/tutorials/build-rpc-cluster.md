---
title: "Build Rpc Cluster"
date: 2021-05-04T16:44:58+08:00
draft: false
---

# 构建ToadOCR RPC服务集群

## 一、安装并启动ETCD load balance center

​	ToadOCR提供了修改过的ECTD安装脚本，可以在[项目仓库](https://github.com/suvvm/ToadOCREngine/blob/master/resources/script/etcd_install.sh)获取。

​	完成安装ETCD后使用[启动脚本](https://github.com/suvvm/ToadOCREngine/blob/master/resources/script/etcd_start.sh)，启动ETCD load balance center

## 二、修改并启动ToadOCREngine server

