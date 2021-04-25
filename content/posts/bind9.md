---
title: bind9(todo detail)
tags:
- devops
date: 2020-08-20 09:54:00
---

```
named-checkconf
named-checkzone yoogo.local zones/yoogo.local
systemctl restart bind9.service
# 一定要重启下网络
netplan apply
```

