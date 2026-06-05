---
title: "Ubuntu 改造 Bash history 格式：显示登录用户和时间"
date: 2026-06-05
draft: false
tags: ["Ubuntu", "Bash", "Linux", "系统配置", "运维"]
categories: ["系统配置"]
description: "让 history 命令显示哪个用户、哪个终端、什么时间执行过该命令，适用于多用户服务器场景。"
---

## 前言

多用户 Linux 服务器上，`history` 默认只显示命令编号和内容，根本不知道是谁、什么时间执行的。排查问题、安全审计时非常不便。

本文记录在 Ubuntu 系统级（所有用户生效）修改 Bash history 格式的方法，让每条记录带上登录用户、终端和精确时间。

<!--more-->

## 官方资源

| 资源 | 地址 |
|------|------|
| **GNU Bash 手册 - History** | <https://www.gnu.org/software/bash/manual/bash.html#Bash-History-Builtins> |
| **Ubuntu 社区文档** | <https://help.ubuntu.com/community/EnvironmentVariables> |

## 最终效果

修改前：

```
 1001  apt update
 1002  systemctl status nginx
 1003  vim /etc/nginx/nginx.conf
```

修改后：

```
 1027  root:pts/0:10.8.0.2 2026-06-05 10:23:45 apt update
 1028  ubuntu:pts/1:192.168.1.100 2026-06-05 11:35:12 systemctl status nginx
 1029  root:pts/0:10.8.0.2 2026-06-05 14:01:30 vim /etc/nginx/nginx.conf
```

一目了然：**谁:终端:来源IP → 时间 → 命令**。

## 配置步骤

### 1. 编辑系统级 bashrc

```bash
sudo vim /etc/bash.bashrc
```

### 2. 在末尾粘贴以下内容

```bash
### History command display extension
HISTFILESIZE=2000
HISTSIZE=2000
USER=$(who -u am i | awk '{print $1 ":" $2 ":" $NF}')
HISTTIMEFORMAT="$USER %Y-%m-%d %H:%M:%S "
export HISTTIMEFORMAT
```

### 3. 生效

```bash
source /etc/bash.bashrc
```

也可以退出重登，或：

```bash
exec bash
```

## 配置详解

### `HISTFILESIZE` / `HISTSIZE`

```bash
HISTFILESIZE=2000
HISTSIZE=2000
```

两个变量的区别：

| 变量 | 作用 |
|------|------|
| `HISTFILESIZE` | `.bash_history` 文件最大行数，重启 shell 后保留 |
| `HISTSIZE` | 当前会话内存中保留的命令数 |

设成一致避免歧义。数值可根据需要调整（服务器建议 `5000`+）。

### `USER` 变量解析

```bash
USER=$(who -u am i | awk '{print $1 ":" $2 ":" $NF}')
```

这是核心。`who -u am i` 输出：

```
ubuntu    pts/0        2026-06-05 11:20   .           192.168.1.100
```

- `$1` → 登录用户名（`ubuntu`）
- `$2` → 终端号（`pts/0`）
- `$NF` → 来源 IP 地址（`192.168.1.100`）

用 `:` 拼接成 `ubuntu:pts/0:192.168.1.100`。

> **注意**：`who -u am i` 始终显示**真实的 SSH 登录用户**，即使通过 `su` 切换了用户。用 `whoami` 或 `$USER` 只能取到当前用户，会丢失原始登录信息。

### `HISTTIMEFORMAT`

```bash
HISTTIMEFORMAT="$USER %Y-%m-%d %H:%M:%S "
```

这是 Bash 内置的格式化变量：

- `%Y` — 四位数年份
- `%m` — 月份（01-12）
- `%d` — 日期（01-31）
- `%H` — 小时（00-23）
- `%M` — 分钟（00-59）
- `%S` — 秒（00-59）

末尾的**空格很重要**，让输出和命令之间有分隔，否则会连在一起。

### 为什么用 `/etc/bash.bashrc` 而非 `~/.bashrc`

| 文件 | 生效范围 | 优先级 |
|------|----------|--------|
| `/etc/bash.bashrc` | **所有用户的交互式 bash** | 系统级，全局生效 |
| `~/.bashrc` | 仅当前用户 | 用户级，可覆盖系统配置 |

多用户服务器建议用 `/etc/bash.bashrc`，**一次配置，所有用户生效**。如果有用户想自定义，可以在自己的 `~/.bashrc` 中覆盖 `HISTTIMEFORMAT`。

## 进阶用法

### 只显示用户和时间，不显示终端和 IP

```bash
USER=$(who -u am i | awk '{print $1}')
HISTTIMEFORMAT="$USER %Y-%m-%d %H:%M:%S "
```

### 加入命令执行耗时（Bash 4.3+）

```bash
HISTTIMEFORMAT="$USER %Y-%m-%d %H:%M:%S [%E] "
```

`%E` 显示命令执行时间。只在 `history` 读取时生效。

### 记录所有命令（含多个终端同时登录）

```bash
# 避免多个 shell 覆盖历史
shopt -s histappend
# 每执行一条命令立即追加到文件，而非退出时才写入
PROMPT_COMMAND="history -a; $PROMPT_COMMAND"
```

配合前面的格式，实现实时无丢失的历史记录。

## 常见问题

### Q1: source 后 history 还是没变化？

可能原因：

- 当前 shell 是 dash、zsh 或其他非 bash shell → `echo $SHELL` 确认
- 如果通过 SSH 执行单条命令（`ssh user@host command`）不经过交互式 bash → 不会读取 bashrc
- 用 `su` 切换用户后 `who -u am i` 可能为空，导致 USER 变量为空 → 检查输出

### Q2: 历史记录能保存多少条？

由 `HISTFILESIZE` 控制。建议服务器设为 `5000-10000`，兼顾查看需要和文件大小：

```bash
HISTFILESIZE=10000
HISTSIZE=10000
```

### Q3: 历史记录文件在哪？

`~/.bash_history`。每个用户独立，默认纯文本存储。

### Q4: 我想让每个用户自己控制，不全局改？

那就把代码写入各用户的 `~/.bashrc`，不在 `/etc/bash.bashrc` 中添加即可。

## 总结

通过修改 `/etc/bash.bashrc` 中的 `HISTTIMEFORMAT` 和自定义 `USER` 变量，可以让 history 命令从这样：

```
 1001  apt update
```

变成这样：

```
 1002  ubuntu:pts/0:192.168.1.100 2026-06-05 11:35:12 apt update
```

**核心配置三行搞定**，建议配合 `shopt -s histappend` 和 `PROMPT_COMMAND` 一起使用，实现全量、实时、可追溯的命令历史记录——多用户服务器运维必备。
