# Windows 上安装 Hermes Agent 完整指南

> Hermes Agent 是 Nous Research 开发的开源 AI Agent，支持多种 LLM 后端。

## 环境要求

| 项目 | 说明 |
|------|------|
| 系统 | Windows 10/11 |
| WSL2 | 2.6.x+，内核 6.6+ |
| Linux 发行版 | Ubuntu 24.04 LTS |
| 网络 | 能访问 GitHub、PyPI |

## 方案一：WSL2 Ubuntu 直接安装（推荐）

### 1. 安装 WSL2 + Ubuntu

```powershell
# PowerShell（管理员）
wsl --install Ubuntu-24.04
```

安装过程中创建用户名和密码。注意：Unix 用户名必须以**小写字母**开头（如 `user9165`，不能用 `9165`）。

### 2. 进入 WSL 并安装依赖

```bash
# 进入 WSL
wsl -d Ubuntu-24.04

# 安装系统依赖
sudo apt update
sudo apt install -y curl ca-certificates git build-essential xz-utils ripgrep ffmpeg
```

### 3. 安装 UV 包管理器

```bash
curl -LsSf https://astral.sh/uv/install.sh | sh
source $HOME/.local/bin/env
```

### 4. 克隆 Hermes Agent 仓库

> 如果在 WSL 内 `git clone` 失败（TLS 错误），从 Windows 宿主机克隆后拷贝。

```bash
# 方法 A：直接在 WSL 内克隆（网络好时）
git clone --depth 1 https://github.com/NousResearch/hermes-agent.git ~/.hermes/hermes-agent

# 方法 B：从 Windows 宿主机拷贝（网络不好时）
# 先在 Windows Git Bash 中：
#   git clone --depth 1 https://github.com/NousResearch/hermes-agent.git /tmp/hermes-agent
# 然后在 WSL 中：
cp -r /mnt/c/Users/<用户名>/AppData/Local/Temp/hermes-agent ~/.hermes/hermes-agent
```

### 5. 安装 Python 依赖

```bash
cd ~/.hermes/hermes-agent

# 创建虚拟环境
uv venv venv --python 3.11

# 安装基础包
export VIRTUAL_ENV="$HOME/.hermes/hermes-agent/venv"
uv pip install -e "."
```

> 如果 PyPI 下载慢，可换清华镜像：
> `uv pip install -e "." --index-url https://pypi.tuna.tsinghua.edu.cn/simple`

### 6. 创建命令链接

```bash
mkdir -p ~/.local/bin
ln -sf ~/.hermes/hermes-agent/venv/bin/hermes ~/.local/bin/hermes
```

### 7. 配置 DeepSeek API

```bash
# 配置文件
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

# API Key（替换为你的 Key）
echo "DEEPSEEK_API_KEY=sk-xxxxxxxx" > ~/.hermes/.env
```

> DeepSeek Anthropic 端点只接受 `deepseek-v4-pro` 或 `deepseek-v4-flash` 作为模型名，不接受 `[1m]` 等后缀。

### 8. 启动

```bash
# 进入 WSL 后启动
wsl -d Ubuntu-24.04
hermes

# 或在 PowerShell 中用绝对路径（不能用 $HOME，会被 PowerShell 吃掉）
wsl -d Ubuntu-24.04 -- bash -c "/home/<用户名>/.local/bin/hermes"

# 单次提问
hermes -z "你的问题"
```

---

## 方案二：Docker 安装

### 1. 拉取镜像并创建容器

```bash
docker run -d --name hermes-agent \
  -v hermes-home:/root/.hermes \
  -v hermes-data:/data \
  docker.m.daocloud.io/library/ubuntu:22.04 \
  tail -f /dev/null
```

### 2. 进入容器安装依赖

```bash
docker exec -it hermes-agent bash

# 换阿里云源（国内加速）
sed -i 's/archive.ubuntu.com/mirrors.aliyun.com/g' /etc/apt/sources.list

apt update && apt install -y curl ca-certificates git build-essential xz-utils ripgrep ffmpeg

curl -LsSf https://astral.sh/uv/install.sh | sh
export PATH="$HOME/.local/bin:$PATH"
```

### 3. 从宿主机拷贝仓库

容器内 git clone 经常因网络问题失败，最可靠的方式是在宿主机克隆后拷贝：

```bash
# Windows Git Bash 中克隆
git clone --depth 1 https://github.com/NousResearch/hermes-agent.git /tmp/hermes-agent

# 拷贝进容器
docker cp /tmp/hermes-agent hermes-agent:/usr/local/lib/hermes-agent
```

### 4. 安装和配置

```bash
# 容器内
export PATH="$HOME/.local/bin:$PATH"
cd /usr/local/lib/hermes-agent
uv venv venv --python 3.11
export VIRTUAL_ENV=/usr/local/lib/hermes-agent/venv
uv pip install -e "."

ln -sf /usr/local/lib/hermes-agent/venv/bin/hermes /usr/local/bin/hermes

# 配置 API（同方案一步骤 7，但路径为 /root/.hermes/）
```

### 5. 启动

```bash
# 交互式
docker exec -it hermes-agent bash -c "export PATH=/root/.local/bin:\$PATH && hermes"

# 单次提问
docker exec hermes-agent bash -c "export PATH=/root/.local/bin:\$PATH && hermes -z '你的问题'"
```

---

## 其他 LLM 后端配置

### Volcengine Agent Plan（火山引擎）

```yaml
providers:
  volcengine:
    name: Volcengine Agent Plan
    api: https://ark.cn-beijing.volces.com/api/plan/v3
    key_env: VOLCENGINE_API_KEY
    default_model: ark-code-latest
    transport: chat_completions
```

### 通用 OpenAI 兼容端点

```yaml
providers:
  custom:
    name: Custom Endpoint
    api: https://your-endpoint.com/v1
    key_env: CUSTOM_API_KEY
    default_model: your-model
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

### Q: 在 PowerShell 中运行 `hermes` 提示 `command not found`

Windows 中 `$HOME` 变量会被 PowerShell 误解析。使用绝对路径：

```powershell
wsl -d Ubuntu-24.04 -- bash -c "/home/<用户名>/.local/bin/hermes"
```

### Q: git clone 失败（TLS/EOF 错误）

WSL 内 Git 连接 GitHub 不稳定时，在 Windows 宿主机（Git Bash）克隆后用 `cp` 或 `docker cp` 拷贝。

### Q: PyPI 下载超时

```bash
# 方案1：增加超时
export UV_HTTP_TIMEOUT=120

# 方案2：使用清华镜像
uv pip install -e "." --index-url https://pypi.tuna.tsinghua.edu.cn/simple
```

### Q: model `deepseek-v4-pro[1m]` 返回 400 错误

DeepSeek API 不接受 `[1m]` 后缀。模型名只能是 `deepseek-v4-pro` 或 `deepseek-v4-flash`。

### Q: WSL 内网络不可达

```bash
sudo bash -c 'echo "nameserver 114.114.114.114" > /etc/resolv.conf'
```

### Q: `cryptography` 包安装失败

网络偶发超时，重新运行安装命令即可（已安装的包会自动跳过）。

### Q: Docker 容器内 `hermes setup` 提示 `No TTY`

容器内无法运行交互式配置向导，直接用 `hermes config set` 命令配置：

```bash
hermes config set model.provider deepseek
hermes config set model.model deepseek-v4-pro
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
