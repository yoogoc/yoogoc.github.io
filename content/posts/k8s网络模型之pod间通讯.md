---
title: k8s网络模型之pod间通讯
tags:
- k8s
- go
- network
date: 2022-07-31 12:00:00
---

本次只学习一下pod间的网络是怎么流通的，暂时不去管pod内多容器是如何共享localhost的。

不同pod之间有两种形式进行访问：同一主机，不同主机。

![k8s-flannel-vxlan](/images/k8s-flannel-vxlan.png)

同一主机下的通讯还是很简单的，cni虚拟网卡就可以搞定了。这里着重讨论在不同主机下的场景。

# 为什么会有pod间通讯的问题

在k8s中，以每个pod为一个原子，会获得一个唯一的ip地址，在同一台主机上，我们有很多种方法来实现不同pod间的通讯，例如虚拟网卡，iptables，ebpf等等。但是在不同主机上，我们就需要引入一个有状态的组件来维持网络，或者说用软件来定义网络（SDN）。

# example分析

假设有虚拟机vm1，vm2，vm1中有pod1，pod2，vm2中有pod3，pod4：

pod1 -> pod2   ------  本地直接路由

pod1 -> pod3   ------ 因为pod的ip地址都是集群内网地址，不能直接通过网卡发送数据包，所以一种经典的方式就是加一层额外的封包（客户端）解包（服务端）的操作，使得pod间丝滑无感的使用集群地址。

这里采用flannel的vxlan模式进行分析。

![flannel-vxlan](/images/flannel-vxlan.png)

1. flanneld会watch所有k8s node，当有增加或者删除事件到达时，会**程序触发**增加/删除 flannel 对应的arp
2. flannel0是一个vxlan虚拟设备，会自动封包/解包 集群网络的数据包



> todo detail。。。。。

