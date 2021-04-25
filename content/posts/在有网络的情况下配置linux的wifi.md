---
title: 在有网络的情况下配置linux的wifi
tags:
- linux
date: 2020-07-24 22:06:00
---

1. `vim /etc/netplan/00-installer-config.yaml`

```
network:
  renderer: NetworkManager
  wifis:
    wlp2s0:
      dhcp4: no
      dhcp6: no
      addresses:
      - 192.168.1.60/24
      gateway4: 192.168.1.1
      nameservers:
        addresses:
        - 223.5.5.5
        - 223.6.6.6
      access-points:
        "yoogo5g":
          password: "123"
```

2. 安装插件

```
sudo apt-get install wpasupplicant network-manager
```

3. 应用

```
netplan generate
netplan apply
```

