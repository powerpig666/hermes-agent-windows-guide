# Windows 上安装 Hermes Agent 完整指南

> Hermes Agent 是 Nous Research 开发的开源 AI Agent，支持多种 LLM 后端。
> 本文针对国内网络环境做了完整的镜像加速和代理适配。

## 目录

- [环境要求](#环境要求)
- [0. 国内网络加速配置（必读）](#0-国内网络加速配置必读)
- [方案一：WSL2 Ubuntu 直接安装（推荐）](#方案一wsl2-ubuntu-直接安装推荐)
- [方案二：Docker Desktop 安装](#方案二docker-desktop-安装)
- [LLM 后端配置速查](#llm-后端配置速查)
- [常见问题](#常见问题)

---

## 环境要求

| 项目 | 说明 |
|------|------|
| 系统 | Windows 10/11 |
| WSL2 | 2.6.x+，内核 6.6+ |
| Linux 发行版 | Ubuntu 24.04 LTS（WSL）/ Ubuntu 22.04（Docker） |
| 网络 | 能访问 GitHub、PyPI（不稳定可用镜像） |

---

## 0. 国内网络加速配置（必读）

在国内网络环境下，直接访问 GitHub、PyPI、Ubuntu 官方源经常超时。以下是全套加速方案，根据使用场景选择。

### 0.1 apt 源换阿里云镜像

```bash
# WSL / Docker 容器内均可使用
sudo sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list
sudo apt update
```

其他可选镜像：

| 镜像站 | sources.list 地址 |
|--------|-------------------|
| 阿里云 | `mirrors.aliyun.com` |
| 清华 | `mirrors.tuna.tsinghua.edu.cn` |
| 中科大 | `mirrors.ustc.edu.cn` |
| 华为云 | `repo.huaweicloud.com` |

### 0.2 PyPI 镜像（uv 使用清华源）

```bash
# 临时使用
uv pip install -e "." --index-url https://pypi.tuna.tsinghua.edu.cn/simple

# 永久配置（写入 uv 配置文件）
mkdir -p ~/.config/uv
cat > ~/.config/uv/uv.toml << 'EOF'
[pip]
index-url = "https://pypi.tuna.tsinghua.edu.cn/simple"
EOF
```

> 其它可选：阿里云 `https://mirrors.aliyun.com/pypi/simple`、腾讯云 `https://mirrors.cloud.tencent.com/pypi/simple`

### 0.3 npm 镜像（安装 agent-browser 时需要）

```bash
# 使用淘宝镜像
npm config set registry https://registry.npmmirror.com
```

### 0.4 Git 代理 & GitHub 加速

```bash
# 方式1：为 GitHub 单独设置代理（如果你有代理）
git config --global http.https://github.com.proxy http://127.0.0.1:7890

# 方式2：取消代理
git config --global --unset http.https://github.com.proxy

# 方式3：用 mirror 前缀克隆（不保证稳定）
# git clone https://ghproxy.com/https://github.com/xxx/yyy.git
# git clone https://gitclone.com/github.com/xxx/yyy.git
```

> **最可靠方式：** 在 Windows 宿主机（Git Bash）克隆，然后 `cp` 或 `docker cp` 进目标环境。宿主机走的是系统代理/直连，比 WSL/容器内的网络栈稳定得多。

### 0.5 Docker 镜像加速

```bash
# 创建/编辑 Docker Desktop 的 daemon.json
# 路径：C:\Users\<用户名>\.docker\daemon.json

{
  "registry-mirrors": [
    "https://docker.m.daocloud.io",
    "https://dockerhub.icu",
    "https://docker.chenby.cn"
  ]
}
```

配置后重启 Docker Desktop。拉取镜像时用镜像前缀：

```bash
# 不用镜像前缀（daemon.json 已全局配置）
docker pull ubuntu:22.04

# 或显式指定镜像前缀
docker pull docker.m.daocloud.io/library/ubuntu:22.04
```

### 0.6 WSL DNS 修复

WSL 偶尔丢失 DNS 导致所有网络不可用：

```bash
# 临时修复
sudo bash -c 'echo "nameserver 114.114.114.114" > /etc/resolv.conf'
sudo bash -c 'echo "nameserver 223.5.5.5" >> /etc/resolv.conf'

# 永久修复（阻止 WSL 自动覆盖 /etc/resolv.conf）
sudo bash -c 'cat > /etc/wsl.conf << EOF
[boot]
systemd=true
[network]
generateResolvConf=false
EOF'
sudo bash -c 'echo "nameserver 114.114.114.114" > /etc/resolv.conf'
```

### 0.7 超时时间设置

```bash
# 安装大包时增加超时（避免 cryptography、ctranslate2 等大包超时）
export UV_HTTP_TIMEOUT=120
```

---

## 方案一：WSL2 Ubuntu 直接安装（推荐）

### 1. 安装 WSL2 + Ubuntu

```powershell
# PowerShell（管理员）
wsl --install Ubuntu-24.04
```

安装过程中创建用户名和密码。注意：Unix 用户名必须以**小写字母**开头（如 `user9165`，不能用 `9165`）。

### 2. 一键安装脚本

> 进入 WSL 后，复制以下脚本一次性完成所有安装步骤。

```bash
#!/bin/bash
set -e

# ---- 0. 网络加速 ----
sudo sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list
echo "nameserver 114.114.114.114" | sudo tee /etc/resolv.conf > /dev/null

# ---- 1. 系统依赖 ----
sudo apt update
sudo apt install -y curl ca-certificates git build-essential xz-utils ripgrep ffmpeg

# ---- 2. UV 包管理器 ----
if ! command -v uv &> /dev/null; then
    curl -LsSf https://astral.sh/uv/install.sh | sh
    source $HOME/.local/bin/env
fi

# ---- 3. 克隆 Hermes Agent ----
if [ ! -d ~/.hermes/hermes-agent ]; then
    git clone --depth 1 https://github.com/NousResearch/hermes-agent.git ~/.hermes/hermes-agent
fi

# ---- 4. Python 虚拟环境 ----
cd ~/.hermes/hermes-agent
uv venv venv --python 3.11
export VIRTUAL_ENV="$HOME/.hermes/hermes-agent/venv"
export UV_HTTP_TIMEOUT=120
uv pip install -e "."

# ---- 5. 命令链接 ----
mkdir -p ~/.local/bin
ln -sf ~/.hermes/hermes-agent/venv/bin/hermes ~/.local/bin/hermes

echo "=== 安装完成 ==="
echo '下一步：配置 API Key（见文档第 7 节）'
```

> 如果第 3 步克隆失败，退出 WSL，在 Windows Git Bash 中克隆后拷贝：
> ```bash
> git clone --depth 1 https://github.com/NousResearch/hermes-agent.git /tmp/hermes-agent
> ```
> 然后回到 WSL：
> ```bash
> cp -r /mnt/c/Users/$USER/AppData/Local/Temp/hermes-agent ~/.hermes/hermes-agent
> ```

### 3. 配置 DeepSeek API

```bash
cat > ~/.hermes/config.yaml << 'EOF'
model:
  provider: deepseek
  model: deepseek-v4-pro
  base_url: ""

providers:
  deepseek:
    name: DeepSeek
    api: https://api.deepseek.com/anthropic
    key_env: DEEPSEEK_API_KEY
    default_model: deepseek-v4-pro
    transport: anthropic_messages
EOF

echo "DEEPSEEK_API_KEY=你的API_KEY" > ~/.hermes/.env
```

> DeepSeek Anthropic 端点只接受 `deepseek-v4-pro` 或 `deepseek-v4-flash` 作为模型名，`[1m]` 等后缀会导致 400 错误。

### 4. 启动

```bash
# 进入 WSL 后启动
wsl -d Ubuntu-24.04
hermes

# 在 PowerShell 中用绝对路径（$HOME 会被 PowerShell 误解析）
wsl -d Ubuntu-24.04 -- bash -c "/home/<用户名>/.local/bin/hermes"

# 单次提问模式
hermes -z "用 Python 写一个冒泡排序"
```

---

## 方案二：Docker Desktop 安装

### 1. 创建容器

```bash
docker run -d --name hermes-agent \
  -v hermes-home:/root/.hermes \
  -v hermes-data:/data \
  docker.m.daocloud.io/library/ubuntu:22.04 \
  tail -f /dev/null
```

### 2. 进入容器，配置镜像源

```bash
docker exec -it hermes-agent bash
```

```bash
# 容器内
sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list
apt update
apt install -y curl ca-certificates git build-essential xz-utils ripgrep ffmpeg
```

### 3. 安装 UV

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"
```

### 4. 获取 Hermes Agent 源码

容器内 `git clone` 极易失败。建议宿主机克隆后拷贝：

```bash
# ---- Windows Git Bash 中 ----
git clone --depth 1 https://github.com/NousResearch/hermes-agent.git /tmp/hermes-agent

# 拷贝进容器
docker cp /tmp/hermes-agent hermes-agent:/usr/local/lib/hermes-agent
```

### 5. 安装依赖

```bash
# 容器内
export PATH="$HOME/.local/bin:$PATH"
cd /usr/local/lib/hermes-agent
uv venv venv --python 3.11
export VIRTUAL_ENV=/usr/local/lib/hermes-agent/venv
export UV_HTTP_TIMEOUT=120
uv pip install -e "."

ln -sf /usr/local/lib/hermes-agent/venv/bin/hermes /usr/local/bin/hermes
```

### 6. 配置和启动

```bash
# 配置 API（参考方案一第 3 步，路径用 /root/.hermes/）

# 交互式
docker exec -it hermes-agent bash -c "export PATH=/root/.local/bin:\$PATH && hermes"

# 单次提问
docker exec hermes-agent bash -c "export PATH=/root/.local/bin:\$PATH && hermes -z '你好'"
```

---

## LLM 后端配置速查

以下均写入 `~/.hermes/config.yaml` 的 `providers:` 段，API Key 写入 `~/.hermes/.env`。

### DeepSeek（Anthropic 兼容端点）

```yaml
model:
  provider: deepseek
  model: deepseek-v4-pro
  base_url: ""

providers:
  deepseek:
    name: DeepSeek
    api: https://api.deepseek.com/anthropic
    key_env: DEEPSEEK_API_KEY
    default_model: deepseek-v4-pro
    transport: anthropic_messages
```

```bash
echo "DEEPSEEK_API_KEY=sk-xxxxxxxx" >> ~/.hermes/.env
```

### DeepSeek（OpenAI 兼容端点）

```yaml
providers:
  deepseek-openai:
    name: DeepSeek
    api: https://api.deepseek.com/v1
    key_env: DEEPSEEK_API_KEY
    default_model: deepseek-chat
    transport: chat_completions
```

### 火山引擎 Agent Plan

```yaml
providers:
  volcengine:
    name: Volcengine Agent Plan
    api: https://ark.cn-beijing.volces.com/api/plan/v3
    key_env: VOLCENGINE_API_KEY
    default_model: ark-code-latest
    transport: chat_completions
```

> API Key 获取：https://console.volcengine.com/ark/ → 开通 Agent Plan

### 阿里云百炼（通义千问）

```yaml
providers:
  qwen:
    name: 通义千问
    api: https://dashscope.aliyuncs.com/compatible-mode/v1
    key_env: DASHSCOPE_API_KEY
    default_model: qwen-plus
    transport: chat_completions
```

### 智谱 GLM

```yaml
providers:
  glm:
    name: 智谱 GLM
    api: https://open.bigmodel.cn/api/paas/v4
    key_env: GLM_API_KEY
    default_model: glm-4-plus
    transport: chat_completions
```

### Moonshot（Kimi）

```yaml
providers:
  kimi:
    name: Moonshot
    api: https://api.moonshot.cn/v1
    key_env: MOONSHOT_API_KEY
    default_model: moonshot-v1-8k
    transport: chat_completions
```

### Anthropic 原生

```yaml
providers:
  anthropic:
    name: Anthropic
    api: https://api.anthropic.com
    key_env: ANTHROPIC_API_KEY
    default_model: claude-sonnet-4-6
    transport: anthropic_messages
```

---

## 常见问题

### Q1: PowerShell 中运行 `hermes` 提示 `command not found`

PowerShell 会吃掉 `$HOME` 变量。用绝对路径：

```powershell
wsl -d Ubuntu-24.04 -- bash -c "/home/<用户名>/.local/bin/hermes"
```

或者先 `wsl -d Ubuntu-24.04` 进入 WSL，再直接运行 `hermes`。

### Q2: WSL 内 git clone 失败（TLS/EOF/HTTP/2 错误）

WSL 的 NAT 网络栈与 GitHub 的连接不稳定。建议在 Windows 宿主机 Git Bash 克隆后拷贝进 WSL：

```bash
# Git Bash 中
git clone --depth 1 https://github.com/NousResearch/hermes-agent.git /tmp/hermes-agent

# WSL 中
cp -r /mnt/c/Users/$USER/AppData/Local/Temp/hermes-agent ~/.hermes/hermes-agent
```

### Q3: PyPI 下载超时或缓慢

```bash
# 方法1：加超时
export UV_HTTP_TIMEOUT=120

# 方法2：换镜像
uv pip install -e "." --index-url https://pypi.tuna.tsinghua.edu.cn/simple

# 方法3：永久配置（见 0.2 节）
```

### Q4: 模型名 `deepseek-v4-pro[1m]` 返回 400 错误

DeepSeek API 只接受 `deepseek-v4-pro` 或 `deepseek-v4-flash`。`[1m]` 是 Claude Code 的上下文窗口标记，不能传给 DeepSeek API。

### Q5: WSL 内完全无法上网

```bash
# 检查 DNS
cat /etc/resolv.conf

# DNS 丢失时修复
sudo bash -c 'echo "nameserver 114.114.114.114" > /etc/resolv.conf'
```

> 如果修复 DNS 后 Hermes 仍然报 `Connection error`，请参阅 **[Hermes Connection Error 完整排障指南](troubleshooting-hermes-connection-error.md)**，其中详细分析了 WSL DNS 失效 + DeepSeek 端点不稳定两个根因及其修复方法。

### Q6: Docker 容器内 `hermes setup` 提示 `No TTY`

容器内无 TTY，无法运行交互式配置向导。绕过方式：

```bash
# 用 config set 代替 setup
hermes config set model.provider deepseek
hermes config set model.model deepseek-v4-pro

# 或直接写配置文件（参考方案一第 3 步）
```

### Q7: 安装 `[all]` extras 报 `python-olm` 找不到

`[all]` extras 包含 Matrix 加密依赖 `python-olm`，某些镜像源版本滞后导致解析失败。用 `-e "."` 安装基础版即可，核心功能完整。

### Q8: Docker Desktop 拉取镜像超时

配置镜像加速器（见 0.5 节），或用 DaoCloud 前缀直接拉取：

```bash
docker pull docker.m.daocloud.io/library/ubuntu:22.04
docker tag docker.m.daocloud.io/library/ubuntu:22.04 ubuntu:22.04
```

---

## 版本信息

- Hermes Agent: v0.13.0（2026.5）
- Python: 3.11.15
- UV: 0.11.x

## 参考链接

- [Hermes Agent GitHub](https://github.com/NousResearch/hermes-agent)
- [DeepSeek API 文档](https://api-docs.deepseek.com)
- [WSL 官方文档](https://learn.microsoft.com/en-us/windows/wsl/)
- [Docker Desktop](https://www.docker.com/products/docker-desktop/)
