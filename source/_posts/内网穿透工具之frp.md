---
title: 内网穿透工具之frp
categories: 工具
date: 2020-06-17 05:10:42
tags:
---

## 简介
frp 是一个可用于内网穿透的高性能的反向代理应用，支持 tcp, udp 协议，为 http 和 https 应用协议提供了额外的能力，且尝试性支持了点对点穿透。
<!--more-->
官方介绍：frp 仍然处于开发阶段，未经充分测试与验证，不推荐用于生产环境。
frp官方中文文档 https://github.com/fatedier/frp/blob/master/README_zh.md
根据对应的操作系统及架构，从 [Release](https://github.com/fatedier/frp/releases) 页面下载最新版本的程序。
将 frps 及 frps.ini 放到具有公网 IP 的机器上。
将 frpc 及 frpc.ini 放到处于内网环境的机器上。

主要介绍两个作用，其余的可自行看官方文档。
## 通过 ssh 访问公司内网机器
**公网机器**修改frps.ini文件
```
[common]
bind_port = 7000
```
启动 frps：
```
./frps -c ./frps.ini          #前台开启，关闭就会失效
nohup ./frps -c ./frps.ini &  #后台开启
```
**内网环境机器**修改 frpc.ini 文件，假设 frps 所在服务器的公网 IP 为 x.x.x.x：
```
[common]
server_addr = x.x.x.x
server_port = 7000
```
```
[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 6000
```
开启运行
```
./frpc -c ./frpc.ini           #前端开启，关闭就会失效
nohup ./frpc -c ./frpc.ini &  #后台开启
```
外网ssh访问内网服务器
`ssh x.x.x.x -p 6000`
## 通过自定义域名访问部署于内网的 web 服务
启动同上方式同上，配置为内容
**公网机器**修改 frps.ini 文件，设置 http 访问端口为 8080：
```
[common]
bind_port = 7000
vhost_http_port = 8080
```
**内网环境机器**修改 frpc.ini 文件，假设 frps 所在的服务器的 IP 为 x.x.x.x，local_port 为本地机器上 web 服务对应的端口, 绑定自定义域名 www.yourdomain.com:
```
[common]
server_addr = x.x.x.x
server_port = 7000
[web]
type = http
local_port = 80
custom_domains = www.yourdomain.com
```
将 www.yourdomain.com 的域名 A 记录解析到 IP x.x.x.x，如果服务器已经有对应的域名，也可以将 CNAME 记录解析到服务器原先的域名。
通过浏览器访问 http://www.yourdomain.com:8080 即可访问到处于内网机器上的 web 服务。
也可以不用域名通过IP访问，主要在于更改端口，下面给出一些配置。
```
[common]
server_addr = x.x.x.x
server_port = 7000

[ssh]
type = tcp
local_ip = 127.0.0.1
local_port = 22
remote_port = 6000

[web]
type = tcp
local_ip = 127.0.0.1
local_port = 80
remote_port = 8081

[RDP]
type = tcp
local_ip = 192.168.1.30 #电脑在局域网中的内网 IP (如是本机，也可使用 127.0.0.1)
local_port = 3389
remote_port = 7001

[DSM]
type = tcp
local_ip = 192.168.1.40 #在局域网中的内网 IP
local_port = 5000
remote_port = 7002

frp+proxifier
[socks_proxy]
type = tcp
remote_port = 9999
plugin = socks5
ssh把远程服务器的流量，变成本地计算机的socks5代理
ssh -f -N -D 127.0.0.1:1080 -oPort=6000 test@192.168.2.1
```