---
title: kubeadm部署k8s(todo)
tags:
- k8s
date: 2021-04-18 15:57:00
---

# 按照官网检查required

https://kubernetes.io/zh/docs/setup/production-environment/tools/kubeadm/install-kubeadm/

此次使用ubuntu2004 kvm作为主机



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

	```
	curl -s https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
	echo "deb https://mirrors.aliyun.com/kubernetes/apt kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

	```

3. 更新apt

   ```
   sudo apt-get update
   sudo apt-get install -y kubelet kubeadm kubectl
   sudo apt-mark hold kubelet kubeadm kubectl
   ```



