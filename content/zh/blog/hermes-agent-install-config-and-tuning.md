---
title: "Hermes Agent 安装、配置与参数调优完整指南"
date: 2026-05-11
draft: false
tags: ["Hermes Agent", "AI Agent", "LLM", "自动化", "Linux", "Cloudflare Pages"]
categories: ["开发工具"]
---

## 前言

Hermes Agent 是由 [Nous Research](https://nousresearch.com) 开源的 AI Agent 框架，可以用自然语言操控你的终端、文件系统、浏览器和消息平台。它支持 20+ LLM 提供商（OpenRouter、Anthropic、OpenAI、DeepSeek、自定义端点等），并且具备**跨会话持久记忆**、**技能自学习**、**多平台网关**（Telegram、Discord、微信等）和**定时任务**等强大功能。

本教程将手把手带你完成 Hermes Agent 的**安装、配置、参数调优**，并部署博客到 **Cloudflare Pages** 实现自动发布。

<!--more-->

## 官方资源

| 资源 | 地址 |
|------|------|
| **官网** | <https://hermes-agent.nousresearch.com> |
| **GitHub** | <https://github.com/NousResearch/hermes-agent> |
| **文档** | <https://hermes-agent.nousresearch.com/docs/> |
| **技能中心** | <https://hermes-agent.nousresearch.com/docs/reference/skills-catalog> |

---

## 一、安装 Hermes Agent

### 方法一：官方安装脚本（推荐）

```bash
# 一键安装
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash

# 安装完成后重新加载 shell 配置
source ~/.bashrc   # 或 source ~/.zshrc
```

安装脚本会自动：
1. 检测 Python 环境（需要 Python 3.10+）
2. 安装到 `~/.hermes/hermes-agent/`
3. 将 `hermes` 命令加入 PATH

### 方法二：通过 pip 安装

```bash
# 使用 pip 安装
pip install hermes-agent

# 或使用 uv（更快）
uv pip install hermes-agent
```

### 方法三：从源码安装（开发者）

```bash
# 克隆仓库
git clone https://github.com/NousResearch/hermes-agent.git ~/.hermes/hermes-agent

# 进入目录并安装
cd ~/.hermes/hermes-agent
pip install -e .
```

### 验证安装

```bash
# 检查版本
hermes --version

# 健康检查（自动检测依赖和配置问题）
hermes doctor
```

---

## 二、初始化配置

### 2.1 交互式配置向导

```bash
# 启动配置向导
hermes setup
```

向导会依次引导你配置：
- **Model（模型）**：选择 LLM 提供商和模型
- **Terminal（终端）**：选择后端（local / docker / ssh）
- **Gateway（网关）**：配置消息平台（Telegram、Discord 等）
- **Tools（工具）**：启用/禁用工具集
- **Agent（Agent）**：设置最大轮次、推理级别等

### 2.2 配置 API Key（以自定义端点为例）

本文使用腾讯混元（Hy3）作为示例，编辑配置文件：

```bash
hermes config edit
```

在 `config.yaml` 中添加：

```yaml
model:
  default: hy3-preview
  provider: tencent_hy_token_plan
  base_url: https://api.lkeap.cloud.tencent.com/plan/v3
  api_key: sk-your-api-key-here
  api_mode: chat_completions

providers:
  tencent_hy_token_plan:
    base_url: https://api.lkeap.cloud.tencent.com/plan/v3
    api_key: sk-your-api-key-here
    api_mode: chat_completions
    model: hy3-preview
    model_display_name: Hy3 preview
```

也可以直接用命令设置：

```bash
hermes config set model.default hy3-preview
hermes config set model.provider tencent_hy_token_plan
hermes config set model.base_url https://api.lkeap.cloud.tencent.com/plan/v3
hermes config set model.api_key sk-your-api-key-here
```

### 2.3 支持的 LLM 提供商一览

| 提供商 | 认证方式 | 环境变量 |
|--------|----------|----------|
| OpenRouter | API Key | `OPENROUTER_API_KEY` |
| Anthropic | API Key | `ANTHROPIC_API_KEY` |
| Nous Portal | OAuth | `hermes auth` |
| OpenAI | API Key | `OPENAI_API_KEY` |
| DeepSeek | API Key | `DEEPSEEK_API_KEY` |
| xAI / Grok | API Key | `XAI_API_KEY` |
| Hugging Face | Token | `HF_TOKEN` |
| 智谱 GLM | API Key | `GLM_API_KEY` |
| MiniMax | API Key | `MINIMAX_API_KEY` |
| 腾讯混元 | API Key | 配置 `base_url` + `api_key` |
| 自定义端点 | 配置文件 | `model.base_url` + `model.api_key` |

---

## 三、核心配置参数详解

配置文件位于 `~/.hermes/config.yaml`，以下是各核心模块的参数说明。

### 3.1 模型与推理

```yaml
model:
  default: hy3-preview          # 默认模型
  provider: tencent_hy_token_plan # 提供商
  base_url: https://...           # API 端点
  api_key: sk-...                # API 密钥
  api_mode: chat_completions     # API 模式

agent:
  max_turns: 60                  # 单次会话最大轮次
  reasoning_effort: medium       # 推理级别：none|minimal|low|medium|high|xhigh
  verbose: false                 # 是否输出详细日志
```

### 3.2 终端配置

```yaml
terminal:
  backend: local                 # local | docker | ssh | modal
  cwd: .                         # 工作目录
  timeout: 180                   # 命令超时（秒）
  docker_mount_cwd_to_workspace: false
  lifetime_seconds: 300          # Docker 容器生命周期
  container_cpu: 1               # 容器 CPU 核数
  container_memory: 5120         # 容器内存（MB）
```

### 3.3 上下文压缩（Token 节省）

```yaml
compression:
  enabled: true                  # 启用上下文压缩
  threshold: 0.5                 # 触发压缩的阈值（50%）
  target_ratio: 0.2              # 压缩后保留比例（20%）
  protect_last_n: 20             # 保护最近 N 条消息不被压缩

prompt_caching:
  enabled: true                  # 启用 prompt caching（最大节省）
  cache_system_prompt: true      # 缓存系统提示
  ttl: 3600                      # 缓存有效期（秒）
```

### 3.4 会话管理

```yaml
session:
  max_history_turns: 3           # 保留最近 N 轮对话（省 token）
  context_strategy: sliding_window # 上下文策略
  max_context_tokens: 3000        # 最大上下文 token 数
  compression_trigger_ratio: 0.4  # 40% 时触发压缩
```

### 3.5 工具配置

```yaml
tools:
  max_tool_result_chars: 768     # 工具输出最大字符数（截断省 token）
  auto_read_large_files: false   # 是否自动读取大文件
  web_extract_only_content: true # 网页提取仅保留正文
```

### 3.6 LLM 输出控制

```yaml
llm_config:
  max_tokens: 768                # 输出最大 token 数
  temperature: 0.2               # 温度（创造性控制）
  auto_model_routing: true       # 自动模型路由
```

### 3.7 记忆系统

```yaml
memory:
  memory_enabled: true           # 启用持久记忆
  user_profile_enabled: true     # 启用用户画像
  memory_char_limit: 1536        # 记忆存储字符上限
  user_char_limit: 960           # 用户画像字符上限
  nudge_interval: 10             # 提醒间隔（轮）
  flush_min_turns: 6             # 最小刷新轮数
  max_memory_retrievals: 1       # 每次仅检索 1 条记忆（省 token）
  max_memory_token_length: 384   # 注入时最大 token 数
```

### 3.8 会话自动重置

```yaml
session_reset:
  mode: both                      # both | idle | schedule
  idle_minutes: 1440              # 空闲 24 小时自动重置
  at_hour: 4                      # 每天凌晨 4 点重置
```

---

## 四、Token 节省优化方案

如果你关注 API 调用成本或 token 消耗，可以按以下方案优化：

### 4.1 核心原则

| 配置项 | 推荐值 | 说明 |
|--------|--------|------|
| `prompt_caching.cache_system_prompt` | `true` | **最大收益** — 系统提示只计费一次 |
| `session.max_history_turns` | `3` | 只保留最近 3 轮对话 |
| `tools.max_tool_result_chars` | `768` | 截断工具输出，避免上下文膨胀 |
| `llm_config.max_tokens` | `768` | 限制输出长度 |
| `memory.max_memory_retrievals` | `1` | 每次只取最相关的 1 条记忆 |
| `compression.target_ratio` | `0.2` | 压缩后保留 20% 内容 |

### 4.2 一键应用省 token 配置

```bash
# 会话相关
hermes config set session.max_history_turns 3
hermes config set session.max_context_tokens 3000
hermes config set session.compression_trigger_ratio 0.4

# 工具输出
hermes config set tools.max_tool_result_chars 768
hermes config set tools.auto_read_large_files false
hermes config set tools.web_extract_only_content true

# LLM 输出
hermes config set llm_config.max_tokens 768
hermes config set llm_config.temperature 0.2

# 记忆系统
hermes config set memory.memory_char_limit 1536
hermes config set memory.user_char_limit 960
hermes config set memory.max_memory_retrievals 1
hermes config set memory.max_memory_token_length 384

# 上下文压缩
hermes config set compression.enabled true
hermes config set compression.threshold 0.5
hermes config set compression.target_ratio 0.2

# Prompt Caching（最大节省）
hermes config set prompt_caching.enabled true
hermes config set prompt_caching.cache_system_prompt true
```

> ⚠️ 以上配置修改后，需要**重启会话**才能生效：CLI 中按 `/reset`，或退出重进。

---

## 五、工具与技能管理

### 5.1 工具集（Toolsets）

```bash
# 列出所有工具及其状态
hermes tools list

# 交互式管理（推荐）
hermes tools

# 启用/禁用指定工具集
hermes tools enable web
hermes tools disable browser
```

| 工具集 | 功能说明 |
|--------|----------|
| `web` | 网页搜索与内容提取 |
| `browser` | 浏览器自动化（Browserbase / Chromium） |
| `terminal` | Shell 命令与进程管理 |
| `file` | 文件读写、搜索、补丁 |
| `code_execution` | 沙箱 Python 执行 |
| `vision` | 图像分析 |
| `skills` | 技能浏览与管理 |
| `memory` | 跨会话持久记忆 |
| `cronjob` | 定时任务管理 |
| `delegation` | 子 Agent 任务委派 |

> 💡 工具变更在**下次新会话**生效（`/reset`），当前会话不会中断。

### 5.2 技能（Skills）

技能是 Hermes 的"经验积累系统"——解决问题后会自动保存为可复用的技能文档。

```bash
# 浏览技能中心
hermes skills browse

# 搜索技能
hermes skills search "github"
hermes skills search "hugo"

# 安装技能
hermes skills install github-auth
hermes skills install hugo-blog

# 查看已安装技能
hermes skills list

# 检查更新
hermes skills check
hermes skills update
```

---

## 六、消息平台网关配置

Hermes 支持在 Telegram、Discord、微信等 10+ 平台上运行同一个 Agent。

### 6.1 安装并启动网关

```bash
# 安装为后台服务
hermes gateway install

# 启动网关
hermes gateway start

# 查看状态
hermes gateway status
```

### 6.2 配置平台（以 Telegram 为例）

```bash
# 交互式配置
hermes gateway setup
```

按提示输入 Telegram Bot Token（从 [@BotFather](https://t.me/BotFather) 获取）。

支持的平台：

| 平台 | 状态 |
|------|------|
| Telegram | ✅ 支持 |
| Discord | ✅ 支持 |
| Slack | ✅ 支持 |
| WhatsApp | ✅ 支持 |
| Signal | ✅ 支持 |
| Matrix | ✅ 支持 |
| 微信 / Weixin | ✅ 支持 |
| Email | ✅ 支持 |
| Feishu / Lark | ✅ 支持 |
| DingTalk | ✅ 支持 |
| Home Assistant | ✅ 支持 |

---

## 七、定时任务（Cron Jobs）

```bash
# 列出所有任务
hermes cron list

# 创建定时任务（每 30 分钟执行一次）
hermes cron create "30m" --prompt "检查服务器状态并汇报"

# 创建定时任务（每天 9 点）
hermes cron create "0 9 * * *" --prompt "生成每日摘要"

# 暂停/恢复任务
hermes cron pause <job_id>
hermes cron resume <job_id>

# 删除任务
hermes cron remove <job_id>
```

---

## 八、斜杠命令速查（会话内）

在交互式会话中输入以下命令：

### 会话控制

| 命令 | 说明 |
|------|------|
| `/new` 或 `/reset` | 新建会话 |
| `/retry` | 重发上一条消息 |
| `/undo` | 删除最后一轮对话 |
| `/title [名称]` | 为会话命名 |
| `/compress` | 手动触发上下文压缩 |
| `/stop` | 终止后台进程 |

### 配置调整

| 命令 | 说明 |
|------|------|
| `/model [名称]` | 查看或切换模型 |
| `/personality [名称]` | 切换人格（kawaii、concise、technical 等） |
| `/reasoning [级别]` | 设置推理级别 |
| `/verbose` | 切换详细输出 |
| `/yolo` | 切换命令免审批模式 |

### 工具与技能

| 命令 | 说明 |
|------|------|
| `/tools` | 管理工具（交互式） |
| `/skills` | 搜索/安装技能 |
| `/skill <名称>` | 加载技能到当前会话 |
| `/cron` | 管理定时任务 |

---

## 九、部署 Hugo 博客到 Cloudflare Pages

### 9.1 准备工作

1. 注册 [Cloudflare](https://dash.cloudflare.com) 账号
2. 将 Hugo 博客推送到 GitHub（本博客已推送至 `lincoln-G-zhang/hugo-skysmoon-blog`）
3. 确保仓库根目录有 `hugo.toml` 配置文件

### 9.2 在 Cloudflare Pages 创建项目

1. 登录 Cloudflare Dashboard → **Workers & Pages**
2. 点击 **Create Application** → **Pages** → **Connect to Git**
3. 授权 GitHub，选择 `hugo-skysmoon-blog` 仓库
4. 配置构建设置：

| 配置项 | 值 |
|--------|-----|
| **Framework preset** | Hugo |
| **Build command** | `hugo --minify` |
| **Build output directory** | `public` |
| **Root directory** | `/`（默认） |
| **Environment variables** | `HUGO_VERSION` = `0.125.0`（或你使用的版本） |

5. 点击 **Save and Deploy**

### 9.3 自动发布工作流程

```bash
# 1. 在本地添加新文章（如本文）
#    文件位于 content/zh/blog/hermes-agent-install-config-and-tuning.md

# 2. 提交并推送
cd ~/hugo-skysmoon-blog
git add .
git commit -m "docs: add hermes agent install and tuning guide"
git push origin master

# 3. Cloudflare Pages 会自动检测推送并触发构建
#    构建完成后，新文章将自动发布到 https://blog.skysmoon.ccwu.cc
```

### 9.4 自定义域名（可选）

在 Cloudflare Pages 项目设置中添加自定义域名：

1. **Pages** → 选择项目 → **Settings** → **Custom domains**
2. 点击 **Set up a domain**，输入 `blog.skysmoon.ccwu.cc`
3. 按提示在 DNS 服务商处添加 CNAME 记录指向 `<项目名>.pages.dev`

---

## 十、常见问题

### Q1：Hermes Agent 和 Claude Code / Codex 有什么区别？

| 特性 | Hermes Agent | Claude Code | OpenAI Codex |
|------|-------------|-------------|--------------|
| 开源 | ✅ 完全开源 | ❌ 闭源 | ❌ 闭源 |
| 多提供商 | ✅ 20+ | ❌ 仅 Anthropic | ❌ 仅 OpenAI |
| 消息平台 | ✅ 10+ | ❌ 仅终端/IDE | ❌ 仅终端 |
| 持久记忆 | ✅ 内置 | ❌ 无 | ❌ 无 |
| 技能系统 | ✅ 自学习 | ❌ 无 | ❌ 无 |
| 定时任务 | ✅ 内置 | ❌ 无 | ❌ 无 |

### Q2：如何降低 API 调用成本？

1. 启用 `prompt_caching.cache_system_prompt: true` —— 最大节省项
2. 设置 `session.max_history_turns: 3` —— 减少上下文
3. 设置 `llm_config.max_tokens: 768` —— 限制输出长度
4. 设置 `tools.max_tool_result_chars: 768` —— 截断工具输出
5. 使用 `memory.max_memory_retrievals: 1` —— 减少记忆注入

### Q3：配置文件修改后不生效？

大部分配置在**会话启动时**读取，修改后需要：
- CLI 中按 `/reset` 新建会话
- 或退出 `hermes` 重新启动
- 网关需要 `hermes gateway restart`

### Q4：如何启用/禁用某个工具？

```bash
hermes tools enable <工具名>
hermes tools disable <工具名>
# 然后 /reset 重启会话
```

### Q5：Hugo 博客推送到 GitHub 后 Cloudflare 没有自动构建？

检查以下几点：
1. 确认推送到了正确的分支（Cloudflare 默认监听 `main` 或 `master`）
2. 检查 Cloudflare Pages 的 **Build settings** 中分支设置
3. 查看 Cloudflare Dashboard 中的构建日志定位错误
4. 确认 `hugo.toml` 在仓库根目录

### Q6：如何彻底卸载 Hermes Agent？

```bash
hermes uninstall
# 手动清理残留配置（可选）
rm -rf ~/.hermes
```

---

## 总结

Hermes Agent 是一个功能强大且高度可定制的开源 AI Agent 框架。通过本教程你学会了：

- ✅ **安装** Hermes Agent（脚本、pip 或源码）
- ✅ **配置** 模型、终端、工具、记忆系统
- ✅ **调优** Token 消耗，降低 API 成本
- ✅ **使用** 技能系统和定时任务
- ✅ **部署** Hugo 博客到 Cloudflare Pages 实现自动发布

推荐下一步：
1. 运行 `hermes skills browse` 探索更多技能
2. 配置消息平台网关，让 Agent 随时随地为你服务
3. 尝试 `hermes cron create` 设置自动化任务

享受 AI Agent 带来的生产力提升吧！🚀
