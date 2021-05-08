---
title: "Etcd gRPC 服务注册与服务发现 Etcd gRPC Register Resolver"
date: 2021-05-08T21:16:00+08:00
draft: false
---

本章主要介绍了ToadOCR中gRPC使用Etcd集群做负载均衡，服务注册与服务发现逻辑。

## 零、理论介绍

​	在开始介绍ToadOCR的RPC分布式微服务体系前，先简单的的介绍一下RPC

​	这里以头条系的微服务体系为例子

​	RPC就是将服务拆分为不同的可以单独部署的Module，通过一个独立控制中心来控制请求的调度。这里的控制中心指的不是一台机器，而是一个概念。真是状况下控制中心是分布在所有机器上的，这些机器共同组成了维系服务正常运行的集群。通过集群的建立可以更方便的执行下一步诸如负载均衡管理，日志系统设计等等...

![rpc_byte](/images/byte-dance-rpc.png)

​	rpc请求单个的生命周期共分为11个部分

- 1 request：rpc客户端在完成请求构造后会将请求交给客户端proxy
- 2 服务发现：客户端proxy成在拿到请求后会联络rpc控制面也就是分布式集群的控制中心以发现存活的服务端
- 3 服务发现：rpc控制面在得到存活的服务端后将节点信息发送给proxy
- 4 request：客户端proxy在得到服务端节点信息后将请求交付到服务端的proxy等待调度
- 5 request：服务端的proxy将请求交付给服务端
- 6 等待调度： 根据当前服务端状态与TCP链接池状态调度建立连接
- 7 建立连接： 客户端与服务端建立点对点的链接
- 8 执行业务逻辑：服务端根据请求信息分配handler处理具体业务逻辑
- 9 responst：服务端将响应交付给服务端proxy
- 10 responst：服务端proxy将响应交付为客户端proxy
- 11 response： 客户端proxy将响应交付给客户端完成单次请求

![rpc_2](/images/rpc_2.png)

![rpc_1](/images/rpc_1.png)

## 一、获取依赖

- 获取gRPC，详见使用讲解中的环境安装
- 获取etcd elient v3 ，详见使用讲解中的环境安装

## 二、服务注册

​	服务注册：主要思路是创建一个lease，put一个前缀（我们头条系一般将其称为P.S.M Product Service Module）的Key(方便服务发现时根据前缀取key对应的值)；然后通过keepAlive续约，并监听keepAlive通道保持在线，如果不监听，etcd会删除这个lease上的key。

### 1、服务信息

- 服务结构设计

```
// 服务信息
type ServiceInfo struct {
    PSM string
    IP   string
}
// 包含服务信息和etcd参数信息
type Service struct {
    ServiceInfo ServiceInfo
    stop        chan error
    leaseId     clientv3.LeaseID
    client      *clientv3.Client
}
```

- 创建一个注册服务

```go
// NewService 创建一个注册服务
func NewService(info ServiceInfo, endpoints []string) (service *Service, err error) {
    client, err := clientv3.New(clientv3.Config{
        Endpoints:   endpoints,
        DialTimeout: time.Second * 200,
    })
    if err != nil {
        log.Fatal(err)
        return nil, err
    }
    service = &Service{
        ServiceInfo: info,
        client:      client,
    }
    return
}
```

### 2、注册服务启动

​	启动后创建lease然后通过keepAlive续约，并循环监听知道收到终止信号

```go
// Start 注册服务启动
func (service *Service) Start() (err error) {
    ch, err := service.keepAlive()
    if err != nil {
        log.Fatal(err)
        return
    }
    for {
        select {
        case err := <-service.stop:
            return err
        case <-service.client.Ctx().Done():
            return errors.New("service closed")
        case resp, ok := <-ch:
            // 监听lease
            if !ok {
                log.Println("keep alive channel closed")
                return service.revoke()
            }
            log.Printf("Recv reply from service: %s, ttl:%d", service.getKey(), resp.TTL)
        }
    }
    return
}
```

### 3、保活

续约lease

```go
func (service *Service) keepAlive() (<-chan *clientv3.LeaseKeepAliveResponse, error) {
    info := &service.ServiceInfo
    key := info.PSM + "/" + info.IP
    val, _ := json.Marshal(info)
    // 创建一个lease
    resp, err := service.client.Grant(context.TODO(), 5)
    if err != nil {
        log.Fatal(err)
        return nil, err
    }
    _, err = service.client.Put(context.TODO(), key, string(val), clientv3.WithLease(resp.ID))
    if err != nil {
        log.Fatal(err)
        return nil, err
    }
    service.leaseId = resp.ID
    return service.client.KeepAlive(context.TODO(), resp.ID)
}
```

### 4、关闭

​	Stop方法将service.stop 置为nil，Start监听到停止信号执行revoke

```go
func (service *Service) Stop() {
    service.stop <- nil
}
func (service *Service) revoke() error {
    _, err := service.client.Revoke(context.TODO(), service.leaseId)
    if err != nil {
        log.Fatal(err)
    }
    log.Printf("servide:%s stop\n", service.getKey())
    return err
}
```

## 三、服务发现

服务发现：通过前缀（P.S.M）取出key对应的values；然后启动一个监听服务监听key的变化



```go
// Resolver 实现grpc的grpc.resolve.Builder接口的Build与Scheme方法
type Resolver struct {
    endpoints []string
    service   string
    cli       *clientv3.Client
    cc        resolver.ClientConn
}

// service 就是上方的PSM endpoints值得是提供活动的集群端点url 如本项目的http://101.201.70.76:2397
func NewResolver(endpoints []string, service string) resolver.Builder {
    return &Resolver{endpoints: endpoints, service: service}
}

// Scheme return etcd schema
func (r *Resolver) Scheme() string {
    // grpc resolver.Register(r)在注册时，会取scheme，如果一个系统有多个grpc发现，就会覆盖之前注册的
    return schema + "_" + r.service
}

// ResolveNow
func (r *Resolver) ResolveNow(rn resolver.ResolveNowOption) {
}

// Close
func (r *Resolver) Close() {
}

// 实现grpc.resolve.Builder接口的方法
func (r *Resolver) Build(target resolver.Target, cc resolver.ClientConn, opts resolver.BuildOption) (resolver.Resolver, error) {
    var err error

    r.cli, err = clientv3.New(clientv3.Config{
        Endpoints: r.endpoints,
    })
    if err != nil {
        return nil, fmt.Errorf("grpclb: create clientv3 client failed: %v", err)
    }

    r.cc = cc

    // go r.watch(fmt.Sprintf("/%s/%s/", schema, r.service))
    go r.watch(fmt.Sprintf(r.service))

    return r, nil
}

func (r *Resolver) watch(prefix string) {
    addrDict := make(map[string]resolver.Address)

    update := func() {
        addrList := make([]resolver.Address, 0, len(addrDict))
        for _, v := range addrDict {
            addrList = append(addrList, v)
        }
        r.cc.NewAddress(addrList)
    }

    resp, err := r.cli.Get(context.Background(), prefix, clientv3.WithPrefix())
    if err == nil {
        for i, kv := range resp.Kvs {
            info := &ServiceInfo{}
            err := json.Unmarshal([]byte(kv.Value), info)
            if err != nil {

            }
            addrDict[string(resp.Kvs[i].Value)] = resolver.Address{Addr: info.IP}
        }
    }

    update()

    rch := r.cli.Watch(context.Background(), prefix, clientv3.WithPrefix(), clientv3.WithPrevKV())
    for n := range rch {
        for _, ev := range n.Events {
            switch ev.Type {
            case mvccpb.PUT:
                info := &ServiceInfo{}
                err := json.Unmarshal([]byte(ev.Kv.Value), info)
                if err != nil {
                    log.Println(err)
                } else {
                    addrDict[string(ev.Kv.Key)] = resolver.Address{Addr: info.IP}
                }
            case mvccpb.DELETE:
                delete(addrDict, string(ev.PrevKv.Key))
            }
        }
        update()
    }
}
```

