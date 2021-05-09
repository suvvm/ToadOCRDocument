---
title: "集群搭建"
date: 2021-05-08T16:58:17+08:00
draft: false
---

# 集群搭建 Cluster Construction

​	本章主要表述将多个服务器构建为一个分布式集群的方法。

ToadOCR分布式集群使用ETCD进行搭建。所有操作的前提是完成本章目录下的环境搭建部分。

## 一、什么是ETCD

​	etcd是一个高度一致的分布式键值存储，它提供了一种可靠的方式来存储需要由分布式系统或机器集群访问的数据。它可以优雅地处理网络分区期间的leader选举，即使在leader节点中也可以容忍机器故障。etcd 内部采用`raft`协议作为一致性算法，etcd 基于 Go 语言实现。

## 二、集群leader

​	leader是通过选举而产生的处理所有数据提交的节点。假设三个节点的集群，三个节点上均运行 Timer（每个 Timer 持续时间是随机的），Raft算法使用随机 Timer 来初始化 Leader 选举流程，第一个节点率先完成了 Timer，随后它就会向其他两个节点发送成为 Leader 的请求，其他节点接收到请求后会以投票回应然后第一个节点被选举为 Leader。

​	成为 Leader 后，该节点会以固定时间间隔向其他节点发送通知，确保自己仍是Leader。有些情况下当 Follower 们收不到 Leader 的通知后，比如说 Leader 节点宕机或者失去了连接，其他节点会重复之前选举过程选举出新的 Leader。

### 创建集群并添加当前节点

- 进入或创建目录/etc/etcd/

```sh
$ sudo mkdir /etc/etcd/
$ cd /etc/etcd/
```

- 创建配置文件etcd.yml

```sh
$ vim etcd.yml
```

- 编辑配置文件

```yaml
name: etcd_toad_ocr														# 节点名称
data-dir: /var/lib/etcd/default.etcd					# 数据文件存储地址
listen-client-urls: http://0.0.0.0:2379				# 对外提供服务的地址
advertise-client-urls: http://{your ipv4 host}:2379,http://localhost:2379	# 此节点客户端URL列表，会通告群集的其余节点
listen-peer-urls: http://0.0.0.0:2380					# 与集群内其他成员之间的通信地址
initial-advertise-peer-urls: http://{your ipv4 host}:2380 # 当前节点对等URL地址，会通告群集的其余节点
initial-cluster: etcd_toad_ocr=http://{your ipv4 host}:2380,etcd_toad_ocr_1=http://{other ipv4 host}:2380,etcd_toad_ocr_2=http://{other ipv4 host}:2380,...						 # 集群中所有节点的信息
initial-cluster-token: etcd_toad_ocr_cluster	# 当前集群的唯一token
initial-cluster-state: new										# 初始集群状态
```

- 启动etcd

```sh
$ etcd --config-file /etc/etcd/etcd.yml
```

当前配置会创建一个token为etcd_toad_ocr_cluster的新集群，创建后选举当前节点为leader并尝试联络其他节点。

## 三、集群member

### 创建新节点并尝试加入leader所在集群

- 进入或创建目录/etc/etcd/

```sh
$ sudo mkdir /etc/etcd/
$ cd /etc/etcd/
```

- 创建配置文件etcd.yml

```sh
$ vim etcd.yml
```

- 编辑配置文件

```yaml
name: etcd_toad_ocr_1													# 节点名称
data-dir: /var/lib/etcd/default.etcd					# 数据文件存储地址
listen-client-urls: http://0.0.0.0:2379				# 对外提供服务的地址
advertise-client-urls: http://{your ipv4 host}:2379,http://localhost:2379	# 此节点客户端URL列表，会通告群集的其余节点
listen-peer-urls: http://0.0.0.0:2380					# 与集群内其他成员之间的通信地址
initial-advertise-peer-urls: http://{your ipv4 host}:2380 # 当前节点对等URL地址，会通告群集的其余节点
initial-cluster: etcd_toad_ocr=http://{leader ipv4 host}:2380,etcd_toad_ocr_1=http://{your ipv4 host}:2380,etcd_toad_ocr_2=http://{other ipv4 host}:2380,...						 # 集群中所有节点的信息
initial-cluster-token: etcd_toad_ocr_cluster	# 当前集群的唯一token
initial-cluster-state: existing								# 初始集群状态
```

- 启动etcd

```sh
$ etcd --config-file /etc/etcd/etcd.yml
```

​	当前配置会创建一个名为为etcd_toad_ocr_1的节点，创建后尝试加入集群etcd_toad_ocr_cluster并联络其他节点。

**重复上述步骤直至所有节点加入目标集群。**

