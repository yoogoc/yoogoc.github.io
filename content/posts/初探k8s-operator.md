---
title: 初探k8s-operator及实现frpc-operator
tags:
- k8s
- go
date: 2022-07-19 10:12:00
---

# operator模式

> Operator 是 Kubernetes 的扩展软件，它利用 [定制资源](https://kubernetes.io/zh-cn/docs/concepts/extend-kubernetes/api-extension/custom-resources/) 管理应用及其组件。 Operator 遵循 Kubernetes 的理念，特别是在[控制器](https://kubernetes.io/zh-cn/docs/concepts/architecture/controller/) 方面。

[官网介绍](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/#motivation)

简单来说，operator是一种通过实现crd和相应的controller来达到拓展k8s的方法。

实现及应用operator的大致流程：

- 定义crd：可以理解为oop中定义一个类
- 实现controller：watch crd，对crd不同的生命周期做个性化处理，也就是实现operator核心的业务逻辑
- 部署controller：可以部署在集群内部，也可以在外部
- 定义crd实例：可以理解为oop中实例化一个类

## frp

frp 是一个专注于内网穿透的高性能的反向代理应用，支持 TCP、UDP、HTTP、HTTPS 等多种协议。可以将内网服务以安全、便捷的方式通过具有公网 IP 节点的中转暴露到公网。

## frp在k8s中的应用

在正常情况下，外部访问k8s有几种方式：

- service的lb和nodeport类型可以暴露服务
- 使用ingress暴露服务

对于个人项目或者其他某种场景，这个集群并不能直接被外部访问，也就是说没有公网ip的情况如何对外服务。首先frp是满足需求的，只需要有一台公网的云服务器。如果没有frpc-operator的情况，那可能需要如下步骤去穿透服务：

1. 部署frpc deployment
2. 将frpc config置于configmap内

每一次暴露或调整新的节点时，就需要调整configmap，删除pod，才能更新完成。

#### 这一步可以进行优化，我们引入一个config-as-code边车容器：

他会监控指定的configmap，当调整配置以后，边车容器自动调用frpc的reload接口，实现不用删除pod即可更新。

需要注意的是，此边车容器需要configmap的一些权限。

#### 为了学习operator，可以将这个操作集封装成为frpc-operator

## frpc-operator

frpc-operator中定义了两种资源：

### client

每一个client就是一个frpc deployment（事实上这里有点问题，如果多个副本会引起proxy重复，应该使用statefulset，将proxy平均分配至每个副本）。示例：

```yaml
apiVersion: frpc.yoogo.top/v1
kind: Client
metadata:
  name: client-sample
spec:
	// 大致与frpc.ini中的配置差不多，只是支持的字段比较贫瘠
  common:
    server_addr: 127.0.0.1
    server_port: 7000
    token:
      value: passwd
```

### Proxy

```yaml
apiVersion: frpc.yoogo.top/v1
kind: Proxy
metadata:
  name: proxy-sample
spec:
  client: client-sample // 必须指定client 名字
  local_addr: 127.0.0.1 // 本地地址
  local_port: "80" // 本地端口
  tcp: 
    remote_port: "20080" //服务端口
```

