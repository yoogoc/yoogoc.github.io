---
title: ss
tags:
- greatwall
date: 2018-11-23 14:10:01
---
1. py环境

```
apt-get install python-pip
pip install --upgrade pip
pip install setuptools
pip install shadowsocks
```
2. 配置文件

`` vim /etc/shadowsocks.json ``


```
{
    "server":"0.0.0.0",
    "server_port":1024,
    "local_address": "127.0.0.1",
    "local_port":1080,
    "password":"mypassword",
    "timeout":300,
    "method":"aes-256-cfb"
}
```
3. 后台运行/停止shadowsocks

`` ssserver -c /etc/shadowsocks.json -d start/stop ``

4. 开机启动

`` vim /etc/rc.local ``

在exit 0前面加上ss的启动命令
add_index "accidents", ["handle_status"], :name => "index_accidents_on_handle_status"
20180918083739
