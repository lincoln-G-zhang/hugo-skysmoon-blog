---
title: "Ubuntu logrotate 日志轮转工具使用指南"
date: 2026-06-10
draft: false
tags: ["Ubuntu", "logrotate", "日志管理", "运维"]
categories: ["运维"]
---

## 前言

logrotate 是 Linux 系统中用于管理日志文件的工具，可以自动轮转、压缩和删除旧日志。本文详细介绍 logrotate 的配置和使用方法，帮助你高效管理系统日志。

<!--more-->

## 一、logrotate 简介

### 什么是 logrotate

logrotate 是一个专门用于日志文件轮转的工具，主要功能包括：

- **日志轮转**：将大日志文件分割成多个小文件
- **日志压缩**：对旧日志进行压缩保存，节省磁盘空间
- **日志删除**：自动删除过期的日志文件
- **日志通知**：轮转后可执行自定义脚本（如通知进程重新打开日志）

### 工作原理

logrotate 通常通过 cron 定时执行（默认每天执行一次），根据配置文件对日志进行轮转处理。

```bash
# 查看 logrotate 的 cron 配置
cat /etc/cron.daily/logrotate

# 查看 logrotate 配置目录
ls /etc/logrotate.d/
```

## 二、配置示例解析

以下是一个典型的 Nginx 日志轮转配置：

```bash
/usr/local/openresty/nginx/logs/*.log {
    daily                    # 每天轮转一次
    rotate 15                # 保留最近 15 份日志
    dateext                  # 使用日期作为日志扩展名
    dateformat -%Y%m%d       # 日期格式：-20240101
    compress                 # 压缩轮转后的日志
    delaycompress            # 延迟压缩（下一次轮转时压缩前一次的日志）
    missingok                # 如果日志文件不存在，不报错
    notifempty               # 如果日志文件为空，不轮转
    sharedscripts            # 共享脚本（只执行一次）
    postrotate               # 轮转后执行的脚本
        [ -s /usr/local/openresty/nginx/logs/nginx.pid ] && kill -USR1 $(cat /usr/local/openresty/nginx/logs/nginx.pid)
    endscript                # 脚本结束标记
}
```

### 配置参数详解

| 参数 | 说明 |
|------|------|
| `daily` | 每天轮转一次 |
| `weekly` | 每周轮转一次 |
| `monthly` | 每月轮转一次 |
| `yearly` | 每年轮转一次 |
| `rotate N` | 保留最近 N 份日志 |
| `dateext` | 使用日期作为扩展名（如 access.log-20240101） |
| `dateformat FORMAT` | 自定义日期格式 |
| `compress` | 使用 gzip 压缩日志 |
| `delaycompress` | 延迟压缩（保留一份未压缩的日志） |
| `missingok` | 文件不存在时不报错 |
| `notifempty` | 文件为空时不轮转 |
| `sharedscripts` | 多个日志文件共享脚本（只执行一次） |
| `prerotate` | 轮转前执行的脚本 |
| `postrotate` | 轮转后执行的脚本 |
| `endscript` | 脚本结束标记 |
| `copytruncate` | 复制文件内容后清空原文件（适用于不支持重新打开日志的进程） |
| `size SIZE` | 当日志文件达到指定大小后轮转（如 size 100M） |
| `maxage DAYS` | 删除超过指定天数的日志 |
| `create MODE OWNER GROUP` | 创建新日志文件时设置权限和所有者 |

## 三、配置文件结构

### 全局配置

主配置文件 `/etc/logrotate.conf`：

```bash
# 查看全局配置
cat /etc/logrotate.conf
```

示例内容：

```bash
# 每周轮转
weekly

# 保留 4 份日志
rotate 4

# 使用日期扩展名
dateext

# 压缩日志
compress

# 包含所有自定义配置
include /etc/logrotate.d/
```

### 自定义配置

将自定义配置放在 `/etc/logrotate.d/` 目录下：

```bash
# 创建自定义配置
sudo vim /etc/logrotate.d/nginx
```

## 四、常用配置示例

### 示例 1：Nginx 日志轮转

```bash
/usr/local/openresty/nginx/logs/*.log {
    daily
    rotate 15
    dateext
    dateformat -%Y%m%d
    compress
    delaycompress
    missingok
    notifempty
    sharedscripts
    postrotate
        [ -s /usr/local/openresty/nginx/logs/nginx.pid ] && kill -USR1 $(cat /usr/local/openresty/nginx/logs/nginx.pid)
    endscript
}
```

### 示例 2：应用程序日志轮转

```bash
/var/log/myapp/*.log {
    daily
    rotate 7
    compress
    delaycompress
    missingok
    notifempty
    create 644 www-data www-data
    postrotate
        systemctl reload myapp
    endscript
}
```

### 示例 3：按大小轮转

```bash
/var/log/largeapp/*.log {
    size 100M              # 达到 100MB 时轮转
    rotate 10
    compress
    missingok
    notifempty
}
```

### 示例 4：使用 copytruncate

适用于不支持重新打开日志的进程：

```bash
/var/log/legacy-app/*.log {
    daily
    rotate 5
    compress
    copytruncate           # 复制后截断原文件
    missingok
    notifempty
}
```

### 示例 5：多日志文件配置

```bash
/var/log/apache2/*.log /var/log/nginx/*.log {
    daily
    rotate 10
    compress
    missingok
    sharedscripts
    postrotate
        systemctl reload apache2
        systemctl reload nginx
    endscript
}
```

## 五、手动执行 logrotate

```bash
# 测试配置文件（不实际执行）
logrotate -d /etc/logrotate.d/nginx

# 强制执行轮转
logrotate -f /etc/logrotate.d/nginx

# 执行指定配置文件
logrotate /etc/logrotate.d/nginx

# 执行所有配置
logrotate /etc/logrotate.conf
```

### 常用选项

| 选项 | 说明 |
|------|------|
| `-d` | 调试模式（不实际执行） |
| `-f` | 强制轮转（忽略时间条件） |
| `-v` | 详细输出 |
| `-s STATEFILE` | 指定状态文件路径 |

## 六、状态文件

logrotate 会记录上次轮转的时间：

```bash
# 查看状态文件
cat /var/lib/logrotate/status

# 状态文件格式
logrotate state -- version 2
"/var/log/nginx/access.log" 2024-1-10-0:0:0
"/var/log/nginx/error.log" 2024-1-10-0:0:0
```

## 七、日志轮转流程

以 Nginx 为例，完整的轮转流程：

1. **检查条件**：判断是否到达轮转时间或文件大小阈值
2. **重命名文件**：将 `access.log` 重命名为 `access.log-20240110`
3. **创建新文件**：创建新的空 `access.log` 文件
4. **发送信号**：向 Nginx 发送 `USR1` 信号，使其重新打开日志文件
5. **压缩日志**：压缩旧日志文件（如果配置了 compress）
6. **清理旧日志**：删除超过保留数量的旧日志

## 八、常见问题

### Q1：日志轮转后进程不写入新日志？

确保在 `postrotate` 脚本中正确发送信号：

```bash
postrotate
    kill -USR1 $(cat /var/run/nginx.pid)
endscript
```

### Q2：如何测试配置是否正确？

使用调试模式测试：

```bash
logrotate -dv /etc/logrotate.d/nginx
```

### Q3：日志文件被删除后如何处理？

配置 `missingok` 参数：

```bash
missingok
```

### Q4：如何查看轮转历史？

查看状态文件：

```bash
cat /var/lib/logrotate/status
```

### Q5：如何自定义轮转时间？

修改 cron 配置：

```bash
# 编辑 cron 配置
sudo vim /etc/cron.d/logrotate

# 例如改为每 6 小时执行一次
0 */6 * * * root /usr/sbin/logrotate /etc/logrotate.conf
```

## 九、最佳实践

1. **按应用分组**：将同一应用的日志放在一个配置文件中
2. **使用 sharedscripts**：多个日志文件共享脚本，避免重复执行
3. **设置合理保留数量**：根据磁盘空间和日志重要性设置 rotate 值
4. **启用压缩**：节省磁盘空间（delaycompress 可保留一份未压缩日志）
5. **测试配置**：使用 `-d` 参数测试配置是否正确
6. **监控日志**：定期检查日志轮转是否正常工作

## 总结

logrotate 是 Linux 系统中不可或缺的日志管理工具，合理配置可以：

- **节省磁盘空间**：自动压缩和清理旧日志
- **便于日志分析**：日志文件大小可控，便于查看和分析
- **避免日志丢失**：进程可以继续写入新日志文件

通过本文的介绍，你应该能够熟练配置和使用 logrotate 管理系统日志了！
