---
title: kvm install
tags:
- linux
date: 2020-07-24 22:06:00
---

# install(ubuntu 20)

## install command

```shell
# 查看cpu是否支持虚拟化，大于0即为支持
grep -Eoc '(vmx|svm)' /proc/cpuinfo
sudo apt update && sudo apt install cpu-checker -y
kvm-ok

sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils virtinst virt-manager
sudo systemctl is-active libvirtd
# 按需调整addresses，nameservers
# 你的网卡名可能不是enp1s0
cat > /etc/netplan/00-installer-config.yaml <<EOF
network:
  bridges:
    kvmbr0:
      interfaces: [enp7s0]
      dhcp4: no
      dhcp6: no
      addresses: [192.168.1.50/24]
      gateway4: 192.168.1.1
      nameservers:
        addresses: [223.5.5.5, 223.6.6.6]
  ethernets:
    enp7s0:
      dhcp4: no
      dhcp6: no
  version: 2
EOF
netplan apply
```

## explain

- qemu-kvm: 为KVM管理程序提供硬件仿真的软件
- libvirt-daemon-system: 配置文件以将libvirt守护程序作为系统服务运行。
- libvirt-clients: 用于管理虚拟化平台的软件。
- bridge-utils: 一组用于配置以太网桥的命令行工具。
- virtinst: 一组用于创建虚拟机的命令行工具。
- virt-manager: 一个易于使用的GUI界面和支持命令行工具，用于通过libvirt管理虚拟机。

### Gotcha

1. 大部分无线网卡是不支持桥接的



# no cloud-init 方式制作虚拟机ubuntu20.04模板机

1. 主机启动template虚拟机
```
virt-install -n u2004-demo --memory memory=8192,currentMemory=2048 --vcpus 2,maxvcpus=10,cpuset=auto -s 30 -c /data/iso/ubuntu20042.iso --hvm --os-type=generic -f /data/img/u2004-demo.img --graphics vnc,listen=0.0.0.0 --force --autostart --network bridge=kvmbr0

virt-install -n u16-demo --memory memory=8192,currentMemory=2048 --vcpus 6,maxvcpus=12,cpuset=2 -s 30 -c /data/iso/u16.iso --hvm --os-type=generic -f /data/img/u16-demo.img --graphics vnc,listen=0.0.0.0 --force --autostart --network bridge=kvmbr0
```

2. 通过vnc连接进入虚拟机，装机，**配置静态ip**

3. 虚拟机基本配置

```shell
# 设置root密码
sudo -i
passwd
# ssh 支持 root 连接, 将PermitRootLogin置为yes
vim /etc/ssh/sshd_config
service ssh restart
# 使虚拟机可以通过virsh console 进入
systemctl start serial-getty@ttyS0 && systemctl enable serial-getty@ttyS0
```

4. 安装ansible

```shell
apt-get update && apt-get upgrade -y && apt-get dist-upgrade -y && apt-get install python2.7
ln -s /usr/bin/python2.7 /usr/bin/python
curl https://bootstrap.pypa.io/get-pip.py --output get-pip.py && python get-pip.py
pip install ansible==2.6.18 netaddr==0.7.19 -i https://mirrors.aliyun.com/pypi/simple/
```
5. 推荐的配置

   a. Vim -> set paste

# 复制虚拟机及需要更改的配置

复制虚拟机,虚拟机配置

```
virt-clone -o u2004-demo -n ceph-1 -f /data/vmdisk/ceph-1.img && \
virt-clone -o u2004-demo -n ceph-2 -f /data/vmdisk/ceph-2.img && \
virt-clone -o u2004-demo -n ceph-3 -f /data/vmdisk/ceph-3.img && \
virt-clone -o u2004-demo -n k8s-n2 -f /data/vmdisk/k8s-n2.img
virsh start k8s-m1 && \
virsh start k8s-m2 && \
virsh start k8s-n1 && \
virsh start k8s-n2

hostnamectl set-hostname harbor && \
cat > /etc/netplan/00-installer-config.yaml <<EOF
network:
  ethernets:
    ens3:
      addresses:
      - 192.168.1.152/24
      gateway4: 192.168.1.1
      nameservers:
        addresses:
        - 192.168.1.150
        - 223.5.5.5
  version: 2
EOF

netplan apply

virsh snapshot-revert k8s-master 1596347792 && \
virsh snapshot-revert k8s-worker1 1596347848 && \
virsh snapshot-revert k8s-worker2 1596347890 && \
virsh snapshot-revert k8s-worker3 1596347927
```

### 为虚拟机增加额外硬盘

```shell
qemu-img create -f qcow2 /data/vmdisk/ceph-attach1.img 20G
```

```xml
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2' cache='none'/>
  <source file='/data/vmdisk/ceph2-attach1.img'/>
  <target dev='vda' bus='virtio'/>
</disk>
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2' cache='none'/>
  <source file='/data/vmdisk/ceph2-attach2.img'/>
  <target dev='vdb' bus='virtio'/>
</disk>
<disk type='file' device='disk'>
  <driver name='qemu' type='qcow2' cache='none'/>
  <source file='/data/vmdisk/ceph2-attach3.img'/>
  <target dev='vdc' bus='virtio'/>
</disk>
```

## 踩过的坑

1. 硬盘容量超过一定百分比会出现虚拟机paused状态，且resume无效，仍是paused

