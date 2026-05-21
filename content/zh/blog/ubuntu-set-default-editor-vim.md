---
title: "设置 Ubuntu 默认编辑器为 Vim（visudo、crontab -e 生效）"
date: 2026-05-21
draft: false
tags: ["Ubuntu", "Vim", "Linux", "系统配置"]
categories: ["系统配置"]
description: "让 visudo、crontab -e、git 等命令默认使用 Vim 编辑器，告别 nano。"
---

## 前言

Ubuntu 默认编辑器通常是 `nano`，但很多用户更习惯 `vim`。然而直接改 `EDITOR` 环境变量往往不生效——`visudo` 和 `crontab -e` 并不都遵守这个变量。本文记录几种可靠方法，确保系统级工具统一使用 Vim。

<!--more-->

## 官方资源

| 资源 | 地址 |
|------|------|
| **Ubuntu 官方文档** | <https://help.ubuntu.com/> |
| **Debian Alternatives 系统** | <https://wiki.debian.org/DebianAlternatives> |
| **vim 官网** | <https://www.vim.org/> |

## 方法一：update-alternatives（推荐，系统级）

Ubuntu/Debian 使用 `alternatives` 系统管理默认程序，这是最规范的方式。

```bash
# 查看当前 editor 指向
sudo update-alternatives --display editor

# 配置默认 editor，会出现交互式选择菜单
sudo update-alternatives --config editor
```

选择列表中 `vim.basic` 或 `vim.gtk3` 对应的编号即可。

如果没有 vim 选项，先安装：

```bash
sudo apt install vim -y
```

验证：

```bash
# 应该输出 /usr/bin/vim.basic 或类似路径
sudo update-alternatives --query editor | grep Value
```

## 方法二：select-editor（强制 crontab 用 vim）

Ubuntu 专门提供了 `select-editor` 命令，用于设置 `crontab -e` 的默认编辑器，比手动改变量更直接。

```bash
# 运行后会出现交互式菜单，选择 vim 对应的编号
select-editor
```

运行效果示例：

```
1. /bin/nano        <---- 当前默认
2. /usr/bin/vim.basic
3. /usr/bin/vim.tiny

Select number: 2
```

选择 `vim.basic` 后，配置会写入 `~/.selected_editor` 文件：

```bash
cat ~/.selected_editor
# 输出示例：
# SELECTED_EDITOR="/usr/bin/vim.basic"
```

> **注意**：`select-editor` 只影响 `crontab -e`，不影响 `visudo` 或其他程序。如需全面覆盖，请配合方法一或方法三使用。

## 方法三：设置 EDITOR 环境变量（用户级）

对 `crontab -e`、`git commit` 等遵守 `EDITOR` 的程序生效。

```bash
# 追加到 ~/.bashrc（bash 用户）
echo 'export EDITOR=vim' >> ~/.bashrc
echo 'export VISUAL=vim' >> ~/.bashrc
source ~/.bashrc

# 如果是 zsh 用户，改为 ~/.zshrc
echo 'export EDITOR=vim' >> ~/.zshrc
echo 'export VISUAL=vim' >> ~/.zshrc
source ~/.zshrc
```

> **注意**：`VISUAL` 优先级高于 `EDITOR`，建议两个都设置。

## 方法三：单独配置 visudo

`visudo` 有自己专用的编辑器配置，修改 `/etc/sudoers`：

```bash
# 不要用 vim 直接编辑 /etc/sudoers！用 visudo：
sudo visudo
```

在文件中添加或修改：

```sudoers
Defaults editor=/usr/bin/vim.basic
# 或
Defaults editor=/usr/bin/vim
```

也可以用环境变量方式（在 sudoers 中）：

```sudoers
Defaults env_editor
```

然后确保 `EDITOR` 环境变量已设置（参考方法二）。

## 各方法生效范围对比

| 方法 | visudo | crontab -e | git commit | nano 替换 |
|------|--------|------------|------------|-----------|
| `update-alternatives` | ✅ | ✅ | ✅ | ✅ 系统全局 |
| `select-editor` | ❌ | ✅ | ❌ | ❌ 仅 crontab |
| `EDITOR` 环境变量 | ❌（除非 sudoers 配了 `env_editor`） | ✅ | ✅ | ❌ 仅当前用户 |
| `sudoers Defaults editor` | ✅ | ❌ | ❌ | ❌ 仅 visudo |

## 验证

```bash
# 验证 crontab 编辑器
crontab -e
# 应该打开 vim，而非 nano

# 验证 visudo 编辑器
sudo visudo
# 应该打开 vim，而非 nano

# 查看当前 EDITOR
echo $EDITOR
# 输出: vim
```

## 常见问题

### Q1: 设置了 EDITOR 但 visudo 还是用 nano？

`visudo` 默认不读取 `EDITOR` 环境变量，需要在 `/etc/sudoers` 中添加：

```sudoers
Defaults env_editor
```

或修改：

```sudoers
Defaults editor=/usr/bin/vim
```

### Q2: update-alternatives 没有 vim 选项？

先安装 vim：

```bash
sudo apt install vim -y
# 然后重新运行
sudo update-alternatives --config editor
```

### Q3: select-editor 和 EDITOR 变量有什么区别？

| 对比 | `select-editor` | `EDITOR` 变量 |
|------|-----------------|----------------|
| 影响范围 | 仅 `crontab -e` | 所有遵守 `EDITOR` 的程序 |
| 配置位置 | `~/.selected_editor` | `~/.bashrc` / `~/.zshrc` |
| 是否需要 sudo | ❌ | ❌ |

### Q4: 只想改当前用户，不影响系统？

用方法二设置 `EDITOR` 和 `VISUAL` 即可，不需要 `sudo`。

## 总结

| 目标 | 推荐方法 |
|------|----------|
| 系统全局生效（所有用户、所有命令） | `sudo update-alternatives --config editor` |
| 仅 crontab 用 vim | `select-editor`（选 vim） |
| 仅当前用户 | 设置 `~/.bashrc` 中 `EDITOR` / `VISUAL` |
| 仅 visudo | 修改 `/etc/sudoers` 的 `Defaults editor` |

最省心的方案是**方法一 + `select-editor` 组合**：系统级用 `update-alternatives` 覆盖大多数场景，`select-editor` 强制 `crontab -e` 用 vim，双保险无死角。
