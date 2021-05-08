---
title: "中国大陆服务器代理 Proxy China"
date: 2021-05-08T16:54:21+08:00
draft: false
---

​	⚠️警告，本项目代理只用于访问受阻断依赖。经由本文章所导致的一切法律责任有读者个人负责！

## 一、为什么需要代理

​	大陆的开发环境不能满足本项目依赖需求，本项目核心使用golang进行开发，使用go mod管理以来，大陆运营商网络没有完成对go getable的支持，且goproxy.cn中存在部分依赖版本hash code验证错误的问题，该部分依赖不经代理无法访问。且大陆各个运营商提供的网络不便于使用git进行版本管理、CI/CD与自动话部署。

## 二、代理的部署

### 1、准备代理服务器

代理服务器标准：

- 服务器位于可以正常访问依赖的网络环境
- 公网IPv4地址
- 充足的带宽（20+Mbps）
- 有域名的A标签指向该服务器IP

### 2、配置代理服务端

#### a.放行端口

​	在提供服务的运营商安全组中添加入站规则，放行目标机器80或443端口并放行其他一个任意可用接口（本章以2021为例）

#### b.iptables 放行端口

放行80端口TCP流量

```sh
$ iptables -A INPUT -p tcp --dport 80 -j ACCEPT
```

放行443端口TCP流量

```sh
$ iptables -A INPUT -p tcp --dport 443 -j ACCEPT
```

放行2021端口TCP流量

```sh
$ iptables -A INPUT -p tcp --dport 2021 -j ACCEPT
```

#### c.搭建伪装站点

​	代理流量在使用时伪装为普通http与https请求，若一直访问目标机器对应端口，被北理工专利强化过的GFW会认出这些代理流量，从而导致代理服务器或端口遭到封禁。若想使代理长期有效，需要将访问代理端口的流量进行重定向，重定向至伪装站点，GFW在探查时会被引导至伪装主页导致其认为当前代理流量是对目标伪装站点的正常访问流量。

​	准备web服务器nginx/caddy/tomcat，都可以，本项目使用caddy

- 安装caddy

```sh
wget -N --no-check-certificate https://raw.githubusercontent.com/iiiiiii1/doubi/master/caddy_install.sh && chmod +x caddy_install.sh && bash caddy_install.sh
```

使用说明

```sh
启动：/etc/init.d/caddy start
停止：/etc/init.d/caddy stop
重启：/etc/init.d/caddy restart
查看状态：/etc/init.d/caddy status
查看Caddy启动日志：tail -f /tmp/caddy.log
```

- 准备伪装站点代码

  将伪装站点index.html及其他依赖文件放置下方目录

```
/usr/local/caddy/camouflage
```

- 编辑caddy配置文件

```sh
$ cd /usr/local/caddy
$ vim Caddyfile
```

Caddyfile内容

```
http://{解析的域名}:2021 {
root /usr/local/caddy/camouflage/
}
```

- 启动caddy服务器

```sh
$ /etc/init.d/caddy start
```

- 访问目标域名加端口确定伪装站点是否正常工作

  本项目伪装站点http://aws.suvvm.work

#### d.配置代理工具服务端

​	本项目使用shadowsocksR作为代理服务端

- 安装shadowsocksR

```sh
$ wget -N --no-check-certificate https://raw.githubusercontent.com/teddysun/shadowsocks_install/master/shadowsocksR.sh && chmod +x shadowsocksR.sh && ./shadowsocksR.sh
```

- 替换配置文件

```sh
$ vim /etc/shadowsocks.json
```

配置如下

```
{
    "server":"0.0.0.0",
    "server_ipv6":"[::]",
    "server_port":80,
    "local_address":"127.0.0.1",
    "local_port":1080,
    "password":"{your password}",
    "timeout":120,
    "method":"none",
    "protocol":"auth_chain_a",
    "protocol_param":"",
    "obfs":"tls1.2_ticket_auth",
    "obfs_param":"",
    "redirect":["*:80#127.0.0.1:2021"],
    "dns_ipv6":false,
    "fast_open":false,
    "workers":1
}
```

上述配置文件使用80端口监听代理流量，并将接收到的代理请求重定向至2021端口，即伪装站点所在位置。使用none加密应用层明文，让国家知道你在干些什么，遵纪守法不加密，做个好公民。

- 重启代理服务端

```sh
$ /etc/init.d/shadowsocks restart
```

- 测试

  http访问目标域名，若可以成功打开伪装站点，则证明代理服务端正常运行

### 3、配置代理客户端

配置被代理主机

- 客户端安装

```sh
$ wget https://raw.githubusercontent.com/the0demiurge/CharlesScripts/master/charles/bin/ssr
$ cp ssr /usr/local/bin/ssr
$ yum install jq
$ ssr install
```

- 配置客户端

```sh
ssr config
```

配置文件内容

```
{
	"server":"{代理主机域名}",
	"server_port": 80,
	"local_address: "127.0.0.1",
	"local_port": 1080,
	"password":"{your password}",
	"method": "none",
	"protocol": "auth_chain_a",
	"protocol_param":"",
  "obfs":"tls1.2_ticket_auth",
  "obfs_param":"",
  "timeout": 120,
  "udp_timeout": 60,
  "dns_ipv6":false,
  "fast_open":false,
}
```

- 配置本机http代理

```sh
$ export http_proxy=http://127.0.0.1:1080
$ export https_proxy=http://127.0.0.1:1080
```

