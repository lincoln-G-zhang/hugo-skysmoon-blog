---
title: "Ubuntu 24.04 systemd 服务性能调优指南"
date: 2026-05-10
draft: false
tags: ["Ubuntu", "systemd", "性能调优", "Linux"]
categories: ["运维"]
---

## 前言

Ubuntu 24.04 默认的 systemd 配置适用于普通桌面场景，但对于高并发服务器、高性能数据库或密集型 Web 服务，往往需要针对性地调整参数。本文全面介绍 systemd 调优、文件描述符限制、DNS 解析优化等关键领域的配置方法。

<!--more-->

## Ubuntu 24.04 systemd 默认限制

Ubuntu 24.04 的 systemd 默认限制定义在系统配置文件中：

```bash
# 查看 systemd 默认资源限制
systemctl show | grep -i defaultlimit
systemctl show --property DefaultLimitNOFILE
systemctl show --property DefaultLimitNPROC

# 也可以直接查看配置文件
grep -i limit /etc/systemd/system.conf
```

| 资源类型 | 默认软限制 | 默认硬限制 | 说明 |
|----------|-----------|-----------|------|
| `LimitNOFILE` | 1024 | 524288 | 文件描述符数量 |
| `LimitNPROC` | 512 | 524288 | 进程/线程数量 |
| `LimitCORE` | infinity | infinity | Core 文件大小 |
| `LimitMEMLOCK` | 65536 | 65536 | 内存锁定大小 |

> 注意：虽然硬限制设置为 524288，但系统级 `fs.file-max` 通常默认为 9223372036854775807（几乎无限制），实际瓶颈往往在 systemd 配置层。

## Root 用户是否需要单独设置？

**是的，root 用户需要单独考虑。**

### 为什么 root 用户特殊？

1. ** PAM 配置不影响 root**：`/etc/security/limits.conf` 中的 `*` 通配符默认**不包含 root 用户**
2. ** systemd 默认包含 root**：systemd 的 `DefaultLimit*` 适用于所有用户，包括 root
3. ** 服务通常以 root 运行**：很多系统服务（nginx、mysql 等）默认以 root 身份运行

### root 用户在 limits.conf 中的写法

```bash
# /etc/security/limits.conf
# 必须显式写 root，不能用 * 包含
root soft nofile 65535
root hard nofile 65535
root soft nproc 65535
root hard nproc 65535
```

### systemd 对 root 服务的配置

```ini
# /etc/systemd/system/nginx.service.d/override.conf
[Service]
LimitNOFILE=65535
LimitNPROC=65535
# 以下配置对 root 用户尤其重要
LimitMEMLOCK=infinity
```

## 全局 systemd 调优配置

### 修改系统级 systemd 配置

编辑 `/etc/systemd/system.conf`：

```ini
[Manager]
# 文件描述符限制
DefaultLimitNOFILE=65535

# 进程数限制
DefaultLimitNPROC=65535

# Core 文件大小（infinity 表示无限制）
DefaultLimitCORE=infinity

# 内存锁定限制（高并发服务需要）
DefaultLimitMEMLOCK=infinity

# CPU 时间限制
DefaultLimitCPU=infinity

# 最大锁数量
DefaultLimitLOCKS=524288
```

### 修改用户级 systemd 配置（可选）

编辑 `/etc/systemd/system.conf.d/limits.conf`：

```ini
[Manager]
DefaultLimitNOFILE=65535
DefaultLimitNPROC=65535
```

### 使配置生效

```bash
# 重载 systemd 配置
sudo systemctl daemon-reload

# 查看当前生效的限制
systemctl show | grep -i limit
```

## 单个服务的限制配置

### 方法一：创建 override 配置文件（推荐）

```bash
# 为服务创建 override 目录
sudo mkdir -p /etc/systemd/system/nginx.service.d/

# 创建 override 配置文件
sudo vim /etc/systemd/system/nginx.service.d/override.conf
```

```ini
[Service]
# 文件描述符
LimitNOFILE=1048576

# 进程数
LimitNPROC=65535

# 内存锁定（对于高性能服务很重要）
LimitMEMLOCK=infinity

# 实时调度（适用于低延迟要求的服务）
LimitRTPRIO=50
LimitRTTIME=infinity

# 最大的文件大小
LimitFSIZE=infinity
```

```bash
# 重载并重启服务
sudo systemctl daemon-reload
sudo systemctl restart nginx

# 验证配置
cat /proc/$(pgrep -f nginx)/limits | grep -E "Max (files|nproc|locks)"
```

### 方法二：直接修改 service 文件

```bash
# 查看服务完整配置
sudo systemctl cat nginx

# 编辑服务文件
sudo systemctl edit nginx --full
```

在 `[Service]` 部分添加：

```ini
[Service]
LimitNOFILE=1048576
LimitNPROC=65535
LimitMEMLOCK=infinity
```

## DNS 域名解析优化

在高并发场景下，DNS 解析失败是常见问题。常见原因包括：

1. **DNS 缓存不足**：系统 DNS 缓存太小
2. **并发连接限制**：DNS 查询并发数受限
3. **DNS 服务器响应慢**：单一 DNS 服务器成为瓶颈
4. **DNS 查询超时**：重试机制不完善

### systemd-resolved DNS 缓存配置

Ubuntu 24.04 使用 `systemd-resolved` 作为 DNS 解析器。配置 `/etc/systemd/resolved.conf`：

```ini
[Resolve]
# 主 DNS 服务器（推荐使用公共 DNS）
DNS=8.8.8.8 1.1.1.1 223.5.5.5

# 备用 DNS
FallbackDNS=8.8.4.4 1.0.0.1

# DNS 缓存大小（默认 1000，严重不足）
CacheSize=10000

# 是否启用 DNSSEC 验证（高并发时可暂时关闭）
DNSSEC=no

# 是否启用 DNS over TLS（生产环境建议开启）
DNSOverTLS=no

# 并发 DNS 查询数
MaxReuseLoopsPerServer=10

# 允许 DNS 服务在缓存耗尽时返回 NXDOMAIN
CacheFromLocalhost=no

# DNS TTL 最小值（秒）
MinTTL=60

# DNS TTL 最大值（秒）
MaxTTL=3600

# 缓存条目-positive TTL 最长时间
CacheMaxTTL=3600

# 缓存条目-negative TTL 最长时间
CacheMinTTL=60
```

```bash
# 重启 systemd-resolved 使配置生效
sudo systemctl restart systemd-resolved

# 验证 DNS 缓存状态
resolvectl statistics
```

### 增加 DNS 并发限制

对于高并发服务，还需要调整内核参数。编辑 `/etc/sysctl.conf`：

```bash
# DNS 查询优化
net.ipv4.tcp_syncookies = 1

# 增加本地端口范围（用于发起大量 DNS 查询）
net.ipv4.ip_local_port_range = 1024 65535

# 调整 ICMP  redirect 忽略设置（安全性考虑）
net.ipv4.conf.all.accept_redirects = 0
net.ipv4.conf.default.accept_redirects = 0
net.ipv6.conf.all.accept_redirects = 0
net.ipv6.conf.default.accept_redirects = 0

# 启用反向路径校验（安全性考虑）
net.ipv4.conf.all.rp_filter = 1
net.ipv4.conf.default.rp_filter = 1
```

```bash
# 使配置生效
sudo sysctl -p
```

### 使用 nscd 进行更强力的 DNS 缓存

如果 systemd-resolved 缓存仍不够，可以安装 nscd（Name Service Cache Daemon）：

```bash
# 安装 nscd
sudo apt install nscd

# 配置 nscd
sudo vim /etc/nscd.conf
```

```conf
# DNS 缓存配置
enable-cache            hosts           yes
positive-time-to-live    hosts           3600
negative-time-to-live     hosts           60
suggested-size            hosts           211
check-files              hosts           yes
persistent               hosts           yes
shared                   hosts           yes
max-db-size              hosts           33554432

# 启用日志（调试时开启）
logfile                 /var/log/nscd.log
debug-level             0
```

```bash
sudo systemctl enable nscd
sudo systemctl restart nscd
```

### 验证 DNS 解析性能

```bash
# 测试 DNS 解析速度
dig example.com

# 测试并发 DNS 查询
for i in {1..100}; do dig example.com +short & done; wait

# 查看 DNS 缓存命中率
nscd -i hosts
getent hosts example.com
nscd -g
```

## 内核网络参数优化

编辑 `/etc/sysctl.conf` 添加以下配置：

```bash
# ========================
# 网络基础优化
# ========================

# 允许更多的并发连接
net.core.somaxconn = 65535
net.core.netdev_max_backlog = 65535
net.core.rmem_max = 16777216
net.core.wmem_max = 16777216
net.core.rmem_default = 262144
net.core.wmem_default = 262144
net.core.optmem_max = 65536

# ========================
# TCP 连接优化
# ========================

# TCP 内存窗口
net.ipv4.tcp_rmem = 4096 87380 16777216
net.ipv4.tcp_wmem = 4096 65536 16777216
net.ipv4.tcp_mem = 786432 1048576 1572864

# TCP 性能参数
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_slow_start_after_idle = 0
net.ipv4.tcp_keepalive_time = 300
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl = 15
net.ipv4.tcp_max_tw_buckets = 262144
net.ipv4.tcp_max_syn_backlog = 65535

# TCP 平滑重启
net.ipv4.tcp_sack = 1
net.ipv4.tcp_fack = 1
net.ipv4.tcp_syncookies = 1

# TIME_WAIT 连接复用
net.ipv4.tcp_tw_reuse = 1

# 本地端口范围（高并发必需）
net.ipv4.ip_local_port_range = 1024 65535

# ========================
# 文件系统优化
# ========================

# 文件描述符
fs.file-max = 2097152
fs.nr_open = 2097152
fs.inotify.max_user_watches = 524288
fs.aio-max-nr = 1048576

# ========================
# 虚拟内存优化
# ========================

vm.swappiness = 10
vm.dirty_ratio = 60
vm.dirty_background_ratio = 5
vm.vfs_cache_pressure = 50
vm.zone_reclaim_mode = 0
vm.max_map_count = 1048576

# ========================
# IPC 共享内存
# ========================

kernel.shmmax = 68719476736
kernel.shmall = 4294967296
kernel.shmmni = 4096
kernel.sem = 250 32000 100 128
```

```bash
# 应用所有 sysctl 配置
sudo sysctl -p

# 验证关键参数
sysctl net.core.somaxconn
sysctl net.ipv4.tcp_tw_reuse
sysctl fs.file-max
```

## PAM 模块配置

确保 `/etc/pam.d/common-session` 包含会话配置：

```bash
# 检查是否存在 pam_limits.so
grep -r "pam_limits" /etc/pam.d/

# 如果没有，添加到 common-session
sudo vim /etc/pam.d/common-session
```

添加行：

```
session required                        pam_limits.so
```

## 完整调优检查清单

### 1. 系统级限制

```bash
# 检查系统最大文件描述符
cat /proc/sys/fs/file-max

# 检查当前使用量
cat /proc/sys/fs/file-nr

# 检查系统最大进程数
cat /proc/sys/kernel/pid_max
```

### 2. systemd 限制

```bash
# 查看默认限制
systemd-analyze show-property DefaultLimitNOFILE
systemd-analyze show-property DefaultLimitNPROC

# 查看所有服务限制
systemctl show --all | grep -E "^Limit(NOFILE|NPROC)="
```

### 3. 用户级限制

```bash
# 查看当前用户限制
ulimit -a

# 查看特定进程限制
cat /proc/$(pgrep nginx)/limits
```

### 4. DNS 解析状态

```bash
# 查看 systemd-resolved 缓存统计
resolvectl statistics

# 查看 nscd 统计（如果使用）
nscd -g
```

## 常见问题排查

### Q1: 修改 limits.conf 后不生效？

```bash
# 确保 PAM 配置包含 pam_limits.so
grep pam_limits /etc/pam.d/*

# 对于 SSH 会话，需要重启 sshd
sudo systemctl restart sshd

# 对于已经登录的 shell，需要重新登录
exit
```

### Q2: systemd 服务 limit 不生效？

```bash
# 确保使用 daemon-reload 而不是 restart
sudo systemctl daemon-reload
sudo systemctl restart your-service

# 检查是否有 override 配置冲突
sudo systemctl cat your-service
```

### Q3: DNS 解析偶尔超时？

```bash
# 增加 DNS 重试和超时
vim /etc/systemd/resolved.conf
# 设置 DNSSEC=no 以减少验证时间
# 设置 CacheSize=10000 增加缓存

# 或者使用 DNS over HTTPS
DNSOverTLS=opportunistic
```

### Q4: 高并发下连接被重置？

```bash
# 检查 TCP 连接跟踪表
cat /proc/sys/net/netfilter/nf_conntrack_max

# 如果使用 iptables，可能需要增加连接跟踪数
echo 1048576 > /proc/sys/net/netfilter/nf_conntrack_max
```

## 总结

Ubuntu 24.04 的 systemd 调优涉及多个层面：

| 层面 | 关键配置 | 说明 |
|------|----------|------|
| **systemd 全局** | `DefaultLimitNOFILE` | 适用所有服务 |
| **systemd 服务** | `LimitNOFILE` | 针对单个服务 |
| **PAM 系统** | `limits.conf` | 影响用户 shell |
| **内核参数** | `sysctl.conf` | 系统级优化 |
| **DNS 缓存** | `resolved.conf` | 域名解析性能 |
| **Root 用户** | 显式指定 | `root soft nofile xxx` |

建议在高并发生产环境中，按照本文档逐步调整，每次修改后监控效果，确保稳定性。
