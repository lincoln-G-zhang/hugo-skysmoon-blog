---
title: "uv Python 包管理工具使用指南"
date: 2026-05-10
draft: false
tags: ["Python", "uv", "包管理", "Linux"]
categories: ["开发工具"]
---

## 前言

uv 是由 [Astral](https://astral.sh)（Rust Python linter `ruff` 的开发团队）开发的极速 Python 包管理工具。相比传统 pip，uv 的安装速度可快 **10-100 倍**，是现代 Python 开发的高效选择。

<!--more-->

## 官方资源

| 资源 | 地址 |
|------|------|
| **官网** | <https://astral.sh/uv> |
| **GitHub** | <https://github.com/astral-sh/uv> |
| **文档** | <https://docs.astral.sh/uv/> |

## Linux 安装 uv

### 方法一：官方安装脚本（推荐）

```bash
# 一键安装
curl -LsSf https://astral.sh/uv/install.sh | sh

# 安装特定版本
curl -LsSf https://astral.sh/uv/0.4.0/install.sh | sh
```

安装脚本会自动：
1. 下载对应平台的二进制文件
2. 安装到 `~/.local/bin/uv`
3. 安装 `uvx`（uv 工具运行器，类似 npx）

### 方法二：使用 pip 安装

```bash
pip install uv
```

### 方法三：使用 conda 安装

```bash
conda install uv -c conda-forge
```

### 方法四：下载二进制文件手动安装

```bash
# 下载最新版本的二进制文件
wget https://github.com/astral-sh/uv/releases/latest/download/uv-x86_64-unknown-linux-gnu.tar.gz

# 解压到指定目录
tar -xzf uv-x86_64-unknown-linux-gnu.tar.gz -C ~/.local/bin/

# 确保在 PATH 中
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

### 验证安装

```bash
# 检查 uv 版本
uv --version

# 检查 uvx 版本
uvx --version
```

## 基础使用方法

### 1. 创建虚拟环境

```bash
# 在当前目录创建 .venv
uv venv

# 指定 Python 版本
uv venv --python 3.12
uv venv --python python3.11

# 指定虚拟环境位置
uv venv .venv

# 使用系统 Python 创建
uv venv --python python
```

### 2. 激活虚拟环境

```bash
# Bash/Zsh
source .venv/bin/activate

# Fish
source .venv/bin/activate.fish
```

### 3. 安装依赖包

```bash
# 安装单个包
uv pip install requests

# 安装多个包
uv pip install requests flask django

# 安装特定版本
uv pip install "requests>=2.28"

# 从 requirements.txt 安装
uv pip install -r requirements.txt

# 安装可编辑模式（开发模式）
uv pip install -e .
```

### 4. 升级和卸载包

```bash
# 升级包
uv pip install --upgrade requests

# 卸载包
uv pip uninstall requests

# 卸载多个包
uv pip uninstall requests flask
```

### 5. 查看已安装的包

```bash
# 列出所有已安装的包
uv pip list

# 检查特定包是否已安装
uv pip show requests

# 检查可升级的包
uv pip list --outdated
```

### 6. 同步依赖

```bash
# 根据 pyproject.toml 安装依赖
uv pip sync

# 指定文件
uv pip sync requirements.txt
```

## 项目管理（推荐方式）

### 创建新项目

```bash
# 创建新项目（自动生成 pyproject.toml）
uv init myproject

# 进入项目目录
cd myproject

# 安装依赖
uv add requests

# 添加开发依赖
uv add --dev pytest black
```

### pyproject.toml 示例

```toml
[project]
name = "myproject"
version = "0.1.0"
description = "My project"
requires-python = ">=3.11"
dependencies = [
    "requests>=2.28",
    "flask>=2.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=7.0",
    "black>=23.0",
]
```

### 运行项目

```bash
# 运行 Python 脚本
uv run python main.py

# 运行已安装的命令
uv run pytest
```

## uv run 临时环境模式

`uv run` 是 uv 的强大特性，可以无需预先创建虚拟环境，直接在临时环境中运行脚本：

```bash
# 直接运行脚本（自动安装依赖）
uv run python main.py

# 运行脚本并指定依赖
uv run --with requests python main.py

# 运行脚本并指定多个依赖
uv run --with requests --with pandas python main.py

# 运行 Python REPL
uv run python

# 运行脚本并指定 Python 版本
uv run --python 3.12 python main.py
```

## 工具运行器（uvx）

`uvx` 类似于 Node.js 的 `npx`，可以快速运行各种命令行工具：

```bash
# 运行 ruff linter
uvx ruff check .

# 运行 black formatter
uvx black .

# 运行 mypy 类型检查
uvx mypy .

# 运行 pytest
uvx pytest

# 指定版本运行
uvx ruff@0.1.0 check .

# 一次性安装并运行多个工具
uvx --from ruff --from black -- ruff check . && black .
```

## 多版本 Python 管理

uv 内置了 Python 版本管理功能：

```bash
# 列出可用的 Python 版本
uv python list

# 安装特定版本
uv python install 3.12
uv python install 3.11.8

# 卸载 Python 版本
uv python uninstall 3.11

# 查找特定版本
uv python find 3.12
```

## 加速 pip 操作

uv 可以作为 pip 的直接替代品：

```bash
# 将 uv 设为默认 pip
uv pip install --system requests

# 安装到系统 Python（需要权限）
sudo uv pip install --system requests

# 或者使用 user 模式
uv pip install --user requests
```

## 镜像源配置

### 临时使用镜像

```bash
# 使用清华镜像
uv pip install requests -i https://pypi.tuna.tsinghua.edu.cn/simple

# 使用阿里镜像
uv pip install requests -i https://mirrors.aliyun.com/pypi/simple/
```

### 全局配置镜像

创建 `~/.config/uv/uv.toml`：

```toml
# 镜像源配置
[[index]]
url = "https://pypi.tuna.tsinghua.edu.cn/simple/"
name = "pypi"
default = true

# 或者使用阿里镜像
# [[index]]
# url = "https://mirrors.aliyun.com/pypi/simple/"
# name = "aliyun"
# default = true
```

## 与 pip、poetry、pipenv 对比

| 特性 | uv | pip | poetry | pipenv |
|------|-----|-----|--------|--------|
| 安装速度 | 极快（Rust） | 慢 | 较慢 | 较慢 |
| 依赖解析 | 快速确定 | 慢 | 慢 | 慢 |
| 锁文件 | uv.lock | 无 | poetry.lock | Pipfile.lock |
| 虚拟环境 | 自动管理 | 手动 | 自动 | 自动 |
| Python 管理 | 内置 | 无 | 无 | 无 |
| 工具运行 | uvx | 无 | 无 | 无 |

## 常见问题

### Q1: uv 和 pip 的区别？

uv 是用 Rust 编写的，专注于极速安装和依赖解析。pip 是 Python 编写的，是官方标准工具。uv 完全兼容 pip 的安装格式，可以作为 pip 的替代品。

### Q2: uv 可以替代 conda 吗？

uv 主要替代 pip，不是 conda。但如果只需要包管理不需要环境管理，uv 是更好的选择。如果需要管理非 Python 依赖（如 C 库），仍需使用 conda。

### Q3: uv 的缓存目录在哪里？

```bash
# 默认缓存目录
~/.cache/uv

# 查看缓存大小
du -sh ~/.cache/uv

# 清理缓存
uv cache clean

# 指定缓存目录
UV_CACHE_DIR=/path/to/cache uv pip install requests
```

### Q4: 如何禁用缓存？

```bash
UV_NO_CACHE=1 uv pip install requests
```

## 完整工作流示例

```bash
# 1. 安装 uv
curl -LsSf https://astral.sh/uv/install.sh | sh

# 2. 创建新项目
uv init myproject
cd myproject

# 3. 添加依赖
uv add requests flask

# 4. 添加开发依赖
uv add --dev pytest black

# 5. 运行开发服务器
uv run flask run

# 6. 运行测试
uv run pytest

# 7. 构建发布
uv build

# 8. 发布到 PyPI
uv publish
```

## 总结

uv 是现代 Python 开发的利器，其核心优势：

- **极速安装**：比 pip 快 10-100 倍
- **统一工具链**：包管理、虚拟环境、Python 版本管理、工具运行
- **完全兼容**：兼容 pip、requirements.txt、pyproject.toml
- **现代化设计**：锁文件、确定性安装、跨平台支持

推荐从今天开始使用 uv 替代 pip，享受极速 Python 开发体验！
