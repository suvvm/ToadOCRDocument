---
title: "使用RPC客户端"
date: 2021-05-04T16:36:37+08:00
draft: false
---

# 使用RPC客户端 Use RPC Client

​	**本章是在完成依赖安装的前提下进行操作**

## go

### 1、在项目下创建目录

- 创建go package rpc 以存放全部rpc客户端

  ```sh
  $ mkdir rpc
  ```

- 创建目录toad_orc_processor_client

  ```sh
  $ mkdir rpc/toad_orc_processor_client
  ```

- 创建目录idl以存放idl文件

  ```sh
  $ mkdir rpc/toad_orc_processor_client/idl
  ```

### 2、根据idl生成客户端代码

- 获取[idl文件](https://github.com/suvvm/ToadOCRPreprocessor/blob/master/rpc/idl/toad_ocr_preprocessor.proto)至rpc/toad_orc_processor_client/idl 目录

- 使用指令生成客户端代码

  ```sh
  $ protoc -I rpc/idl rpc/toad_orc_processor_client/idl/toad_ocr_preprocessor.proto --go_out=plugins=grpc:rpc/idl
  ```

- 处理生成的代码

  ```sh
  $ sed -i '' 's/ClientConnInterface/ClientConn/g' rpc/toad_orc_processor_client/idl/toad_ocr_preprocessor.pb.go

  $ sed -i '' 's/SupportPackageIsVersion6/SupportPackageIsVersion4/g' rpc/toad_orc_processor_client/idl/toad_ocr_preprocessor.pb.go
  ```

### 3、创建服务发现文件

​	在rpc/toad_orc_processor_client目录下创建go文件 resolver.go并添加以下内容，服务注册与发现相见“分布式集群”部分

```go
package rpc
import (
	"context"
	"fmt"
	"strings"

	"github.com/coreos/etcd/clientv3"
	"github.com/coreos/etcd/mvcc/mvccpb"
	"google.golang.org/grpc/resolver"
)

const schema = "etcdv3_resolver"

// resolver is the implementaion of grpc.resolve.Builder
type Resolver struct {
	target  string
	service string
	cli     *clientv3.Client
	cc      resolver.ClientConn
}

// NewResolver return resolver builder
// target example: "http://127.0.0.1:2379,http://127.0.0.1:12379,http://127.0.0.1:22379"
// service is service name
func NewResolver(target string, service string) resolver.Builder {
	return &Resolver{target: target, service: service}
}

// Scheme return etcdv3 schema
func (r *Resolver) Scheme() string {
	return schema
}

// ResolveNow
func (r *Resolver) ResolveNow(rn resolver.ResolveNowOption) {
}

// Close
func (r *Resolver) Close() {
}

// Build to resolver.Resolver
func (r *Resolver) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOption) (resolver.Resolver, error) {
	var err error

	r.cli, err = clientv3.New(clientv3.Config{
		Endpoints: strings.Split(r.target, ","),
	})
	if err != nil {
		return nil, fmt.Errorf("grpclb: create clientv3 client failed: %v", err)
	}

	r.cc = cc

	go r.watch(fmt.Sprintf("/%s/%s/", schema, r.service))

	return r, nil
}

func (r *Resolver) watch(prefix string) {
	addrDict := make(map[string]resolver.Address)

	update := func() {
		addrList := make([]resolver.Address, 0, len(addrDict))
		for _, v := range addrDict {
			addrList = append(addrList, v)
		}
		r.cc.UpdateState(resolver.State{Addresses: addrList})
	}

	resp, err := r.cli.Get(context.Background(), prefix, clientv3.WithPrefix())
	if err == nil {
		for i := range resp.Kvs {
			addrDict[string(resp.Kvs[i].Value)] = resolver.Address{Addr: string(resp.Kvs[i].Value)}
		}
	}

	update()

	rch := r.cli.Watch(context.Background(), prefix, clientv3.WithPrefix(), clientv3.WithPrevKV())
	for n := range rch {
		for _, ev := range n.Events {
			switch ev.Type {
			case mvccpb.PUT:
				addrDict[string(ev.Kv.Key)] = resolver.Address{Addr: string(ev.Kv.Value)}
			case mvccpb.DELETE:
				delete(addrDict, string(ev.PrevKv.Key))
			}
		}
		update()
	}
}

```

### 4、编写客户端内容

在rpc/toad_orc_processor_client 目录下创建文件client.go

这里提供调用测试接口Ping的代码

```go
package rpc

import (
	"context"
	"flag"
	"google.golang.org/grpc"
	"google.golang.org/grpc/balancer/roundrobin"
	"google.golang.org/grpc/resolver"
	"io/ioutil"
	"log"
	"suvvm.work/ToadOCRPreprocessor/common"
	pb "suvvm.work/ToadOCRPreprocessor/rpc/idl"
	"time"
)

func RunRpcClient() {
	flag.Parse()
	r := NewResolver(*reg, *serv)
	resolver.Register(r)
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	conn, err := grpc.DialContext(ctx, r.Scheme()+"://authority/"+*serv, grpc.WithInsecure(), grpc.WithBalancerName(roundrobin.Name), grpc.WithBlock())
	defer cancel()
	if err != nil {
		panic(err)
	}
	defer conn.Close()
	client := pb.NewToadOcrPreprocessorClient(*conn)
	resp, err := client.Ping(ctx, &pb.PingRequest{Name: "test"})
  if err != nil {
    log.Printf("error %v", err)
	}
  log.Printf("ping msg:%v", resp.Message)
}

```

### 5、Main函数调用客户端发送请求

```
func main() {
	rpc.RunRpcClient()
}
```

