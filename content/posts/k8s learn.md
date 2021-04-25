---
title: k8s基本操作
tags:
- k8s
date: 2020-05-20 22:06:00
---

## 删除节点
```shell
# 主机执行
kubectl delete node k8s-worker-x
# node机执行
sudo kubeadm reset
```

  ### 常用查看命令

```shell
kubectl get nodes
kubectl describe node xxx
```

### 好用的配置

```shell
# 永久保存该上下文中所有后续 kubectl 命令使用的命名空间
kubectl config set-context --current --namespace=<insert-namespace-name-here>
```



## 踩到的坑

1. 莫名ingress映射的host访问不通
2. node单节点ping不通：`sudo /etc/init.d/networking restart`



## thinking

###万物皆容器

### Master 关键进程：


1. Kubernetes API Server (kube-apiserver)：提供了HTTP Rest接口的关键服务进程，是Kubernetes里所有资源的增、删、改、查等操作的唯一入口，也是集群控制的入口进程
2. Kubernetes Controller Manager (kube-controller-manager): Kubernetes里所有的资源对象的自动化控制中心
3. Kubernetes Scheduler (kube-scheduler)：负责资源调度(Pod调度)的进程
4. etcd: Kubernetes里的所有**资源对象**的数据全部是保存在`etcd`中

### node 关键进程

1. **kubelet：**负责Pod对应容器的创建、停止等任务，同时与Master节点密切协作，实现集群管理的基本功能
2. **kube-proxy：**实现Kubernetes Service的通信与负载均衡机制的重要组件。
3. **Docker Engine(Docker)：**Docker引擎，负责本机的容器创建和管理工作。



### 知识点

1. 每创建一个 [Service](https://kubernetes.io/docs/user-guide/services) 时，会创建一个相应的 [DNS 条目](https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/)， 访问形式：`<service-name>.<namespace-name>.svc.cluster.local`

2. 如果pvc在某些条件下删除以后一直处于terminating，则很有可能是因为存在finalizers，尝试清空`kubectl patch pvc db-pv-claim -p '{"metadata":{"finalizers":null}}'`，应该会被删除掉

