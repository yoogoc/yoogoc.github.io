---
title: kubeadm部署k8s
tags:
- k8s
date: 2021-04-18 15:57:00
---

# 按照官网检查required

https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

此次使用ubuntu2004 kvm作为主机

## 检查iptables

```sh
# 检查是否加载模块br_netfilter
lsmod | grep br_netfilter
# 开启
sudo modprobe br_netfilter
```

iptables 能够正确地查看桥接流量：

```sh
cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
EOF
sudo sysctl --system

swapoff -a && sysctl -w vm.swappiness=0
sed -ri '/^[^#]*swap/s@^@#@' /etc/fstab

apt remove -y ufw lxd lxd-client lxcfs lxc-common
```

## 安装 runtime

目前可用的运行时：

| 运行时     | 域套接字                        |
| ---------- | ------------------------------- |
| Docker     | /var/run/dockershim.sock        |
| containerd | /run/containerd/containerd.sock |
| CRI-O      | /var/run/crio/crio.sock         |

> 如果同时检测到 Docker 和 containerd，则优先选择 Docker

这里使用containerd：

```sh
sudo apt-get update
# 会依赖安装runc
sudo apt-get install containerd
containerd config default | sudo tee /etc/containerd/config.toml
```

结合 runc 使用 systemd cgroup 驱动，在 /etc/containerd/config.toml 中设置

```sh
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
    
[plugins."io.containerd.grpc.v1.cri".registry]
  ...
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors]
    [plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
      endpoint = ["https://30l6yhjq.mirror.aliyuncs.com"]
```

设置启动文件

```sh
wget https://github.com/containerd/containerd/archive/v1.5.7.tar.gz
tar xvf v1.5.7.tar.gz && cd containerd-1.5.7
cp containerd.service /etc/systemd/system/
```

启动

```sh
systemctl daemon-reload
systemctl enable containerd.service
systemctl start containerd.service
systemctl status containerd.service
```



# 安装 kubeadm、kubelet 和 kubectl

1. 更新 `apt` 包索引并安装使用 Kubernetes `apt` 仓库所需要的包：

```
sudo apt-get update && \
sudo apt-get install -y apt-transport-https ca-certificates curl
```

2. 添加apt仓库

   a. 可以访问外网的情况：

   ```
   sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
   echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
   ```

   b. 不能访问外网，使用阿里源：

   ```sh
   curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
   cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
   deb https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main
   EOF
   ```

   开源社源（我用404）：

   ```sh
   curl -s https://mirror.azure.cn/kubernetes/packages/apt/doc/apt-key.gpg | sudo apt-key add -
   cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
   deb http://mirror.azure.cn/kubernetes/packages/apt/ kubernetes-xenial main
   EOF
   ```

3. 更新apt

   ```
   sudo apt-get update
   sudo apt-get install -y kubelet kubeadm kubectl
   sudo apt-mark hold kubelet kubeadm kubectl
   ```

4. 创建kubeadm config

```sh
kind: ClusterConfiguration
apiVersion: kubeadm.k8s.io/v1beta3
# kubernetesVersion: v1.22.3
imageRepository: registry.aliyuncs.com/google_containers
controlPlaneEndpoint: '192.168.1.141:6443'

apiServer:
  extraArgs:
    authorization-mode: Node,RBAC
  timeoutForControlPlane: 4m0s

networking:
  serviceSubnet: '10.96.0.0/12' # default
  podSubnet: "10.244.0.0/16"
  dnsDomain: 'cluster.local' # default

controllerManager: {}

etcd:
  local:
    dataDir: /var/lib/etcd

certificatesDir: /etc/kubernetes/pki

---
kind: InitConfiguration
apiVersion: kubeadm.k8s.io/v1beta3
nodeRegistration:
  kubeletExtraArgs:
    pod-infra-container-image: 'registry.aliyuncs.com/google_containers/pause:3.5'
    network-plugin: cni
    v: '9'

---
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: ipvs

---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
#cgroupDriver: cgroupfs
```

5. 初始化

```sh
kubeadm init --config kubeadm-init-config.yaml -v 20
```

6. 等待初始化完成,完成后检查k8s状态

```sh
# 获取组件状态
kubectl get cs
```

> 因为kubeadm会默认不开启scheduler和controller-manager的http接口，因此可能会Unhealthy，解决：注释掉`/etc/kubernetes/manifests/kube-scheduler.yaml`,`/etc/kubernetes/manifests/kube-controller-manager.yaml` 中`port=0`字段，重启kubelet

```sh
# 获取节点状态
kubectl get node
kubectl describe nodes
```

此时节点是NotReady状态的，原因可以通过`kubectl describe nodes`中Conditions的得到：

```sh
Conditions:
  Type             Status  LastHeartbeatTime                 LastTransitionTime                Reason                       Message
  ----             ------  -----------------                 ------------------                ------                       -------
  MemoryPressure   False   Sat, 30 Oct 2021 01:19:42 +0000   Fri, 29 Oct 2021 16:21:57 +0000   KubeletHasSufficientMemory   kubelet has sufficient memory available
  DiskPressure     False   Sat, 30 Oct 2021 01:19:42 +0000   Fri, 29 Oct 2021 16:21:57 +0000   KubeletHasNoDiskPressure     kubelet has no disk pressure
  PIDPressure      False   Sat, 30 Oct 2021 01:19:42 +0000   Fri, 29 Oct 2021 16:21:57 +0000   KubeletHasSufficientPID      kubelet has sufficient PID available
  Ready            False   Sat, 30 Oct 2021 01:19:42 +0000   Fri, 29 Oct 2021 16:21:57 +0000   KubeletNotReady              container runtime network not ready: NetworkReady=false reason:NetworkPluginNotReady message:Network plugin returns error: cni plugin not initialized
```

因为没有cni插件，所以会NotReady



7. 安装最先进的cni插件-cilium

```sh
curl -L --remote-name-all https://github.com/cilium/cilium-cli/releases/latest/download/cilium-linux-amd64.tar.gz{,.sha256sum}
sha256sum --check cilium-linux-amd64.tar.gz.sha256sum
sudo tar xzvfC cilium-linux-amd64.tar.gz /usr/local/bin
rm cilium-linux-amd64.tar.gz{,.sha256sum}

cilium install

# 等待安装完成
cilium status
kubectl get node
```



