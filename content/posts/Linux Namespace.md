---
title: Linux Namespace
tags:
- linux
date: 2021-05-09 11:29:00
---

# 什么是linux namespace

命名空间是Linux内核的一个功能，它对内核资源进行分区，使一组进程看到一组资源，而另一组进程看到另一组资源。该功能的作用是为一组资源和进程提供相同的命名空间，但这些命名空间指的是不同的资源。资源可能存在于多个空间。这类资源的例子有：进程ID、主机名、用户ID、文件名，以及一些与网络访问和进程间通信相关的名称。

> https://en.wikipedia.org/wiki/Linux_namespaces



# 现有种类


## 1. UTS--CLONE_NEWUTS

UTS Namespace 主要用来隔离 nodename 和 domainname 两个系统标识。在 UTS Namespace 里面 ， 每个 Namespace 允许有自己的 hostname

## 2. Interprocess Communication (ipc)--CLONE_NEWIPC

IPC Namespace 用来隔离 System V IPC 和 POSIX message queues。 每一个 IPC Namespace 都有自己的 System V IPC 和 POSIX message queue。

## 3. Process ID (pid)--CLONE_NEWPID

PID Namespace是用来隔离进程 ID的。同样一个进程在不同的 PIDNamespace里可以拥 有不同的 PID。这样就可以理解， 在 container里面， 使用 ps-ef经常会发现， 在容器 内， 前台运行的那个进程 PID 是 l， 但是在容器外，使用 ps -ef会发现同样的进程却有不同的PID， 这就是 PIDNamespace做的事情。

> 实验时要注意，要用 `echo $$`来打印进程id，而不是ps，因为ps是读取`/proc`中的内容来显示，但此时`/proc`还是宿主机的内容

## 4. Mount (mnt)--CLONE_NEWNS

MountNamespace用来隔离各个进程看到的挂载点视图。在不同Namespace的进程中， 看 到的文件系统层次是不一样的。在 Mount Namespace 中调用 mount()和 umount()仅仅只会影响 当前 Namespace 内的文件系统 ，而对全局的文件系统是没有影响的。

Mount Namespace 是 Linux 第 一个实现 的 Namespace 类型 ， 因此，它的系统调用参数 是 NEWNS ( New Namespace 的缩 写)。设计者没有预料到会有其他的命名空间。

## 5. User ID (user) --CLONE_NEWUSER

User N amespace 主要是隔离用户 的用户组 ID。 也就是说 ， 一个进程的 User ID 和 Group ID在UserNamespace内外可以是不同的。 比较常用的是，在宿主机上以一个非root用户运行 创建一个 User Namespace， 然后在 User Namespace 里面却映射成 root 用户。这意味着 ， 这个 进程在 User Namespace 里面有 root权限，但是在 User Namespace 外面却没有 root 的权限。从 Linux Kernel 3.8开始， 非root进程也可以创建UserNamespace， 并且此用户在Namespace里 面可以被映射成 root， 且在 Namespace 内有 root权限。

## 6. Network (net)--CLONE_NEWNET

Network Namespace 是用来隔离网络设备、 IP地址端口 等网络械的 Namespace。 Network Namespace 可以让每个容器拥有自己独立的(虚拟的)网络设备，而且容器内的应用可以绑定 到自己的端口，每个 Namespace 内的端口都不会互相冲突。在宿主机上搭建网桥后，就能很方 便地实现容器之间的通信，而且不同容器上的应用可以使用相同的端口 。

## 7. Control group (cgroup) Namespace--CLONE_NEWCGROUP

cgroup命名空间类型隐藏了进程作为成员的控制组的身份。在这样的命名空间中的进程，在检查任何进程属于哪个控制组时，会看到一个实际上是相对于创建时设置的控制组的路径，隐藏其真实的控制组位置和身份

## 8. Time Namespace-- CLONE_NEWTIME

时间命名空间允许进程以类似于UTS命名空间的方式看到不同的系统时间。

# 提案种类

## syslog namespace

