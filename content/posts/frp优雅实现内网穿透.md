---
title: frp优雅实现内网穿透
tags:
- "devops"
date: 2020-01-31 14:10:01
---

## 使用场景

公司有虚拟机可以用,但总不能在家一直挂着vpn吧,一直挂着也可以,但是vpn跟梯子是冲突的,只能开一个,这可咋整,之前用过钉钉的内网穿透服务,感觉很nice,但是现在不给随便用了,没办法,自己搭一个吧

## 前期准备

1. 一台有公网ip的服务器, 以下称为 A
2. 一台内网服务器, 以下称为 B
3. 优秀且简单的轮子: [frp](https://github.com/fatedier/frp) , [releases](https://github.com/fatedier/frp/releases) ,[wiki](https://github.com/fatedier/frp/wiki)

## Fuck it!

1. 两台服务器都下载各自对应版本的frp, A需要使用frps, frps.ini,B需要使用 frpc, frpc.ini
2. 相应配置(自己可以去readme看啥意思)

```ini
# frps.ini
[common]
bind_port = 7000
token = xxxx
```



```ini
# frpc.ini
[common]
server_addr = x.x.x.x
server_port = 7000
token = xxx

[pg]
type = tcp
local_ip = 127.0.0.1
local_port = 5432
remote_port = 5432

[rabbitmq-server]
type = tcp
local_ip = 127.0.0.1
local_port = 5672
remote_port = 5672

[rabbitmq-dashboard]
type = tcp
local_ip = 127.0.0.1
local_port = 15672
remote_port = 15672

[kibana]
type = tcp
local_ip = 127.0.0.1
local_port = 5601
remote_port = 5601

[mongo]
type = tcp
local_ip = 127.0.0.1
local_port = 27017
remote_port = 27017

[elasticsearch1]
type = tcp
local_ip = 127.0.0.1
local_port = 9200
remote_port = 9200

[elasticsearch1]
type = tcp
local_ip = 127.0.0.1
local_port = 9300
remote_port = 9300


```

3. 后台启动

```sh
sudo vim /lib/systemd/system/frps.service
```

```ini
[Unit]
Description=frps service
After=network.target syslog.target
Wants=network.target

[Service]
Type=simple
#启动服务的命令（此处写你的frps的实际安装目录）
ExecStart=/your/path/frps -c /your/path/frps.ini

[Install]
WantedBy=multi-user.target
```

```sh
sudo systemctl start frps
sudo systemctl restart frps
sudo systemctl stop frps
sudo systemctl status frps
```

4. 按照官方service路径轻修改

```shell
wget https://github.com/fatedier/frp/releases/download/v0.34.3/frp_0.34.3_linux_amd64.tar.gz
tar -zxvf frp_0.34.3_linux_amd64.tar.gz frp && cd frp
cp systemd/frpc.service /lib/systemd/system/
cp frpc /usr/bin/
vim frpc.ini
cp frpc.ini /etc/frp/
service frpc start
```

