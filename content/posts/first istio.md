---
title: first istio(待完善)
tags:
- k8s
date: 2020-11-27 23:53:00
---

# 基本概念

### VirtualService

介于service负载均衡到deploy之间，使目标service可控，进而可以达到蓝绿部署/金丝雀部署的效果

模板例子

```yaml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: jason
    route:
    - destination:
        host: reviews
        subset: v2
  - route:
    - destination:
        host: reviews
        subset: v1

```

##### 解释

1. hosts：虚拟服务的主机。虚拟服务主机名可以是 IP 地址、DNS 名称，或者依赖于平台的一个简称（例如 Kubernetes 服务的短名称），隐式或显式地指向一个完全限定域名（FQDN）。您也可以使用通配符（“*”）前缀，让您创建一组匹配所有服务的路由规则。虚拟服务的 `hosts` 字段实际上不必是 Istio 服务注册的一部分，它只是虚拟的目标地址。这让您可以为没有路由到网格内部的虚拟主机建模。

2. http: http协议下的路由规则

（待续···）
