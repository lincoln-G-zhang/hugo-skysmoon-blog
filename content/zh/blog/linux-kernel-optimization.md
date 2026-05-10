---
title: "Linux 内核参数优化指南"
date: 2026-05-10
draft: false
tags: ["Linux", "内核", "优化", "性能"]
categories: ["运维"]
---

## 前言

Linux 系统性能调优是一个持续优化的过程。本文介绍常用的内核参数优化方法，适用于 Web 服务器、数据库服务器等场景。

<!--more-->

## 文件描述符限制

文件描述符是 Linux 系统的重要资源，高并发服务容易耗尽。

### 检查当前限制

```bash
# 查看当前系统限制
ulimit -n

# 查看最大文件描述符
cat /proc/sys/fs/file-max
```

### 永久修改

编辑 `/etc/security/limits.conf`：

```bash
* soft nofile 65535
* hard nofile 65535
* soft nproc 65535
* hard nproc 65535
```

编辑 `/etc/sysctl.conf`：

```bash
fs.file-max = 65535
```

使配置生效：

```bash
sysctl -p
```

## 网络参数优化

### TCP 连接优化

```bash
# 允许更多的 TCP 连接
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 65535

# TCP 内存参数
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216

# 复用 TIME_WAIT 连接
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15

# 保持连接参数
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl = 15
```

### 连接跟踪优化

```bash
# NF 连接跟踪表大小
net.netfilter.nf_conntrack_max = 1048576
net.netfilter.nf_conntrack_tcp_timeout_established = 7200
```

## 内存参数优化

### 共享内存

```bash
# 共享内存大小
kernel.shmmax = 68719476736
kernel.shmall = 4294967296
```

### 消息队列

```bash
# 消息队列参数
kernel.msgmnb = 65536
kernel.msgmax = 65536
```

## 内核调度优化

```bash
# 虚拟内存参数
vm.swappiness = 10
vm.dirty_ratio = 60
vm.dirty_background_ratio = 5

# 文件系统缓存
vm.vfs_cache_pressure = 50
```

## 生效配置

将参数写入 `/etc/sysctl.conf` 后执行：

```bash
# 使配置立即生效
sysctl -p

# 查看所有配置
sysctl -a

# 查看特定参数
sysctl net.ipv4.tcp_tw_reuse
```

## 监控工具

```bash
# 查看系统资源使用
top
htop

# 查看网络连接状态
ss -s
netstat -an | grep ESTABLISHED | wc -l

# 查看文件描述符使用
lsof | wc -l
cat /proc/sys/fs/file-nr
```

## 总结

内核参数优化需要根据实际业务场景和硬件配置进行调整。建议：

1. 每次修改少量参数，便于定位问题
2. 记录每次修改前的值，便于回滚
3. 在测试环境验证后再应用到生产环境
4. 定期监控系统性能指标，持续优化
