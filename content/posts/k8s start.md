---
title: ubuntu下k8s安装
tags:
- k8s
date: 2020-05-20 22:06:00
---

## base: 在主机、和节点需要执行（保证虚拟机/物理机内网地址稳定）

```shell
sudo vim /etc/hostname # k8s-xx，目的：看起来好看
sudo vim /etc/hosts # x.x.x.x k8s-xx，目的：爽
sudo vim /etc/fstab  # 将swap注释掉，目的：禁用交换分区
sudo reboot -h
```



## docker

```shell
sudo apt install docker.io
sudo vi /etc/docker/daemon.json
```

#### 加速docker

执行`sudo vi /etc/docker/daemon.json` 输入：

> 自己去[阿里云容器镜像服务](https://cr.console.aliyun.com/)申请自己的加速地址

```json
{
  "registry-mirrors": [
    "https://docker.mirrors.ustc.edu.cn",
    "http://hub-mirror.c.163.com"
  ],
  "max-concurrent-downloads": 10,
  "log-driver": "json-file",
  "log-level": "warn",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
    },
  "data-root": "/var/lib/docker"
}
```

```shell
sudo systemctl daemon-reload
sudo systemctl restart docker
sudo systemctl enable docker
```



## 安装 kubeadm、kubelet 和 kubectl

```shell
sudo apt-get update && sudo apt-get install -y apt-transport-https curl
# 国内不能用curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo curl https://mirrors.aliyun.com/kubernetes/apt/doc/apt-key.gpg | sudo apt-key add -
# cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
# deb https://apt.kubernetes.io/ kubernetes-xenial main
# EOF
cat <<EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb https://mirrors.aliyun.com/kubernetes/apt/ kubernetes-xenial main
EOF
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```





## init master

坑点：

1. Init 需要拉镜像，网上的例子有好多竟然是把镜像拉下来然后retag解决，着实不优雅，换个镜像地址就搞定了
2. 阿里云可能同步的没有那么快，可以通过手动指定版本解决，事实上生产环境也不会去拉latest

```shell
# apiserver-advertise-address 更换成自己局域网地址
sudo kubeadm init \
--apiserver-advertise-address=192.168.1.220 \
--image-repository registry.cn-hangzhou.aliyuncs.com/google_containers \
--pod-network-cidr=10.244.0.0/16 \
--ignore-preflight-errors=NumCPU \
--kubernetes-version=v1.18.3
# 安装成功以后在worker节点执行：
sudo kubeadm join 192.168.1.220:6443 --token pck7jp.l1l0oq33ue8c40eq \
    --discovery-token-ca-cert-hash sha256:e4f6763e5a9a2d7aa290cf6380a5ee3000ed2a367ca81a373d66308fef44cd36
```



### config kubectl

```shell
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```



### config network

```shell
kubectl apply -f https://docs.projectcalico.org/v3.15/manifests/calico.yaml
```



### lb

```
ansible-playbook roles/ex-lb/ex-lb.yml
```



### rancher

```bash
# rancher 2.5以后会自启一个k3s单节点，因此必须要开特权模式
sudo docker run -d --restart=unless-stopped \
-p 80:80 -p 443:443 \
-v /var/lib/rancher/:/var/lib/rancher/ \
-v /root/var/log/auditlog:/var/log/auditlog \
-e CATTLE_SYSTEM_CATALOG=bundled \
-e AUDIT_LEVEL=3 \
--name rancher \
--privileged=true \
rancher/rancher

rm -rf /var/lib/rancher/ && rm -rf /root/var/log/auditlog
```



# ansible安装

1.  不能通过二进制安装docker，[issues](https://github.com/easzlab/kubeasz/issues/848)

```shell
vim /etc/ssh/sshd_config
service ssh restart
ssh-keygen -t rsa -b 2048 -N '' -f ~/.ssh/id_rsa
ssh-copy-id $IPs #$IPs为所有节点地址包括自身，按照提示输入yes 和root密码
curl https://bootstrap.pypa.io/get-pip.py --output get-pip.py
python get-pip.py

export release=2.2.1
curl -C- -fLO --retry 3 https://github.com/easzlab/kubeasz/releases/download/${release}/easzup
chmod +x ./easzup
./easzup -D
```



helm

```shell
curl https://helm.baltorepo.com/organization/signing.asc | sudo apt-key add -
sudo apt-get install apt-transport-https --yes
echo "deb https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list
sudo apt-get update
sudo apt-get install helm
# 或者
wget https://get.helm.sh/helm-v3.2.1-linux-amd64.tar.gz # 自己去找版本号
mv ./linux-amd64/helm /usr/bin
```

## debug

1. 网络、dns

```
kubectl run -it --rm --restart=Never --image=busybox busybox
kubectl run -it --rm --restart=Never --image=infoblox/dnstools:latest dnstools

=> nslookup kubernetes.default.svc.cluster.local
```

[CoreDNS pods DNS resolution issue]: https://medium.com/@mohanpedala/coredns-pods-dns-resolution-issue-2282110a0b37	"medium.com"



## k3s

占用资源少，但是坑还是很多的，亮点是使用sqlite代替etcd,以及自带local storage class，一定要看安装指南，不然都不知道坑（feature）是哪里来的，比如Service Load Balancer（slb）

https://rancher.com/docs/k3s/latest/en/networking/

https://rancher.com/docs/k3s/latest/en/installation/install-options/server-config/

### Quick start

```shell
curl -sfL https://get.k3s.io | INSTALL_K3S_EXEC="--disable=servicelb" sh -
```

如果网络不好可以先将二进制k3s下载至本地

```shell
# 自己修改指定的版本号
wget https://github.com/rancher/k3s/releases/download/v1.20.4%2Bk3s1/k3s
chmod 755 k3s & mv k3s /usr/local/bin/
curl -sfL https://get.k3s.io | INSTALL_K3S_SKIP_DOWNLOAD=true sh -
cp /var/lib/rancher/k3s/agent/etc/containerd/config.toml /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl
cat >> /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl << EOF
[plugins.cri.registry.mirrors]

[plugins.cri.registry.mirrors."docker.io"]
  endpoint = ["https://30l6yhjq.mirror.aliyuncs.com", "https://docker.mirrors.ustc.edu.cn"]
EOF

systemctl restart k3s
cp /etc/rancher/k3s/k3s.yaml ~/.kube/config

cat /var/lib/rancher/k3s/server/node-token
```

```sh
# cri 加速
cat >> /etc/rancher/k3s/registries.yaml <<EOF
mirrors:
  "docker.io":
    endpoint:
    	- "https://30l6yhjq.mirror.aliyuncs.com"
    - "https://docker.mirrors.ustc.edu.cn"
EOF
systemctl restart k3s


```



### Add node

```shell
curl -sfL http://rancher-mirror.cnrancher.com/k3s/k3s-install.sh |
INSTALL_K3S_MIRROR=cn K3S_URL=https://192.168.1.91:6443 K3S_TOKEN=K10f5c3ae2cd90c834a10eae8cee2225e5b9b250d64187ffd235552b77b14da365f::server:903e9a1b0b622a686dfa85710f09c848 sh -

cp /var/lib/rancher/k3s/agent/etc/containerd/config.toml /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl
cat >> /var/lib/rancher/k3s/agent/etc/containerd/config.toml.tmpl << EOF
[plugins.cri.registry.mirrors]

[plugins.cri.registry.mirrors."docker.io"]
  endpoint = ["https://30l6yhjq.mirror.aliyuncs.com", "https://docker.mirrors.ustc.edu.cn"]
EOF
systemctl restart k3s-agent.service
```

