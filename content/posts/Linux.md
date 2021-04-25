---
title: Linux
tags:
- linux
date: 2018-11-23 14:10:01
---
# namespace

namespace是一种隔离机制，一个独立的namespace看上去拥有所有linux主机的资源，也拥有自己的0号进程（即系统初始化的进程）。一个namespace可以产生多个子namespace，通过设置clone系统调用的flag可以实现。事实上namespace是为了支持linux container（即linux容器）出现的，运用kernel中的namespace机制和cgroup机制（kernel的配额管理机制）可以实现轻量级的虚拟，即多个虚拟主机（容器）公用宿主机的kernel，彼此之间资源隔离。docker的部分技术也依赖于此。

# opt

1. CPU占用最多的前10个进程：
   `ps auxw|head -1;ps auxw|sort -rn -k3|head -10 `

2. 内存消耗最多的前10个进程
   `ps auxw|head -1;ps auxw|sort -rn -k4|head -10 `

3. 虚拟内存使用最多的前10个进程

`ps auxw|head -1;ps auxw|sort -rn -k5|head -10`

4. 命令行关闭显示器

`setterm --blank 1`

