# Hermes Agent 性能优化指南

> 本文涵盖模型选择、上下文压缩、工作流加速、并行模式等提升 Hermes Agent 效率的实用技巧。

## 目录

- [1. 模型选择策略](#1-模型选择策略)
- [2. 上下文压缩调优](#2-上下文压缩调优)
- [3. CLI 启动加速](#3-cli-启动加速)
- [4. 会话管理优化](#4-会话管理优化)
- [5. 并行工作模式](#5-并行工作模式)
- [6. 系统级加速](#6-系统级加速)
- [7. 进阶技巧](#7-进阶技巧)

---

## 1. 模型选择策略

### 1.1 选对模型，大任务用强模型，小任务用快模型

```bash
# 复杂编码/架构任务 — 强模型
hermes -m deepseek-v4-pro

# 简单查询/翻译/摘要 — 快模型（更快更便宜）
hermes -m deepseek-v4-flash

# 临时覆盖本次会话的模型
hermes --model deepseek-v4-flash
```

### 1.2 配置 Fallback 自动故障转移

当主模型不可用时自动切换备用模型：

```bash
# 添加 fallback 提供者
hermes fallback add        # 交互式选择

# 或直接写入 config.yaml
hermes config set fallback '[{"provider":"deepseek","model":"deepseek-v4-flash","api_mode":"anthropic_messages","base_url":"https://api.deepseek.com/anthropic"}]'
```

### 1.3 Max Turns 控制单次对话长度

避免 Agent 陷入无限循环拉高 token 消耗：

```bash
hermes config set model.max_turns 50   # 默认 90，简单任务设为 30-50
```

---

## 2. 上下文压缩调优

Hermes 内置上下文压缩，长对话超过阈值后自动压缩历史消息，避免超出模型上下文窗口。

```bash
# 查看当前压缩配置
hermes config | grep -A5 "Context Compression"

# 调优参数
hermes config set compression.threshold 0.6        # 60% 上下文使用率时触发压缩（默认 50%）
hermes config set compression.target_ratio 0.25    # 压缩后保留 25% 的阈值消息（默认 20%）
hermes config set compression.protect_last 15      # 保护最近 15 条消息不被压缩（默认 20）
hermes config set compression.model deepseek-v4-pro # 压缩用模型（默认 auto = 同主模型）
```

| 参数 | 建议值 | 说明 |
|------|--------|------|
| `threshold` | 0.5–0.7 | 太高会频繁触发压缩，太低浪费 token |
| `target_ratio` | 0.2–0.3 | 保留一定比例确保连续性 |
| `protect_last` | 10–20 | 最近消息最可能被引用，务必保留 |
| `model` | 与主模型相同即可 | 用 flash 模型压缩可进一步节省成本 |

> 也可以完全关闭：`hermes config set compression.enabled false`

---

## 3. CLI 启动加速

### 3.1 Oneshot 模式（最轻量）

单次提问不启动完整 REPL，适合脚本集成：

```bash
# 无 banner、无 spinner、无 session 持久化
hermes -z "解释这段代码" > result.txt
hermes -z "$(cat prompt.txt)"
```

### 3.2 跳过不需要的加载项

```bash
# 跳过 AGENTS.md / SOUL.md / 记忆 / 预加载技能
hermes --ignore-rules

# 跳过用户级 config.yaml，只用内置默认
hermes --ignore-user-config

# 跳过 shell hook 确认（无 TTY 环境）
hermes --accept-hooks
```

### 3.3 按需加载工具集

完整工具集加载会影响启动速度，按需指定：

```bash
# 只加载代码相关工具
hermes -t code_execution,terminal,file

# 不加文件系统工具的场景
hermes -t clarify,code_execution

# 可用的 toolsets: code_execution, terminal, file, browser, web, memory,
#                 skills, todo, clarify, vision, delegation, tts, kanban
```

### 3.4 YOLO 模式（信任环境内免确认）

在本地开发环境中跳过危险命令确认提示：

```bash
hermes --yolo
```

> 仅在可信代码库中使用，生产环境不推荐。

---

## 4. 会话管理优化

### 4.1 长任务用会话恢复

避免每次从头开始，节省 token：

```bash
# 列出过往会话
hermes sessions list

# 按 ID 恢复
hermes --resume <session_id>

# 按名称恢复
hermes -c "项目名称"

# 恢复最近的会话
hermes -c
```

### 4.2 定期清理旧会话

```bash
# 交互式浏览和删除
hermes sessions browse

# 按数量删除旧的
hermes sessions prune --keep 20
```

### 4.3 给关键会话加标签便于恢复

```bash
hermes sessions rename <session_id> "修复登录Bug"
```

---

## 5. 并行工作模式

### 5.1 Worktree 隔离模式

每个 worktree 实例有独立的 git 工作区、会话和记忆，互不干扰：

```bash
# 启动独立 worktree 会话
hermes -w

# 适合：同时处理多个功能分支、并行跑多个 Agent 实例
```

### 5.2 多 Profile 并行

创建多个 profile，每个有独立配置和记忆：

```bash
# 创建 profile
hermes profile create backend-dev

# 在该 profile 中启动
hermes profile backend-dev
hermes

# 列出所有 profile
hermes profile list
```

### 5.3 多窗口并行工作流

```bash
# 窗口1：处理前端代码
wsl -d Ubuntu-24.04 -- bash -c "/home/user/.local/bin/hermes --worktree"

# 窗口2：处理后端代码
wsl -d Ubuntu-24.04 -- bash -c "/home/user/.local/bin/hermes --worktree"
```

---

## 6. 系统级加速

### 6.1 基础工具预装确认

以下工具直接影响执行速度：

| 工具 | 作用 | 安装 |
|------|------|------|
| ripgrep (`rg`) | 文件搜索比 grep 快 10–100 倍 | `sudo apt install ripgrep` |
| Node.js 22+ | 浏览器工具运行时 | 安装器自动装到 `~/.hermes/node/` |
| ffmpeg | TTS 语音消息 | `sudo apt install ffmpeg` |
| agent-browser | 网页自动化 | `npm install -g agent-browser` |

### 6.2 验证工具可用性

```bash
hermes doctor | grep -E "✓|✗"
```

### 6.3 TUI 模式（现代终端界面）

默认是经典 REPL，换成 TUI 响应更快：

```bash
hermes --tui
```

---

## 7. 进阶技巧

### 7.1 管道组合

```bash
# 代码审查管道
cat src/main.py | hermes -z "审查这段代码，只列出 Bug" > review.txt

# 批量翻译
for f in *.md; do
    hermes -z "翻译以下内容为中文: $(cat "$f")" > "zh_CN/$f"
done
```

### 7.2 环境变量控制

```bash
# 模型覆盖（不修改 config）
export HERMES_INFERENCE_MODEL=deepseek-v4-flash

# 提供者覆盖
export HERMES_INFERENCE_PROVIDER=deepseek
```

### 7.3 减少首次对话延迟的技巧

```bash
# 先做一次空跑热身
hermes -z "ping" > /dev/null

# 之后正式使用
hermes
```

### 7.4 使用 Dashboard（Web UI）

Dashboard 提供可视化会话管理、多标签并发：

```bash
# 启动 Dashboard
hermes dashboard

# 浏览器打开 http://localhost:9119

# 查看状态
hermes dashboard --status

# 关闭
hermes dashboard --stop
```

### 7.5 监控 Agent 日志

```bash
# 实时跟踪
hermes logs -f

# 只看错误
hermes logs errors

# 最近 1 小时
hermes logs --since 1h
```

---

## 性能优化速查表

| 场景 | 建议 |
|------|------|
| 快速问答 | `hermes -z` + `-m deepseek-v4-flash` |
| 大规模重构 | `hermes -w`（worktree 隔离）+ `--yolo` |
| 长对话 | 调整 `compression.threshold = 0.5` |
| 多任务并行 | 多窗口 + `--worktree` |
| 脚本集成 | `hermes -z` + stdout 重定向 |
| 减少启动开销 | `--ignore-rules --toolsets code_execution,terminal` |
| 免确认执行 | `--yolo`（仅本地可信环境） |
| 跨分支开发 | `hermes profile create` 隔离环境 |
