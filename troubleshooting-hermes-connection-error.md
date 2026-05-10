# Hermes Connection Error 完整排障指南

> 现象：Hermes 启动后持续报 `APIConnectionError`，重试 3 次后 fallback 也失败，无法正常使用。

## 错误日志特征

```
⚠️  API call failed (attempt 1/3): APIConnectionError
   🔌 Provider: deepseek  Model: deepseek-v4-pro
   🌐 Endpoint: https://api.deepseek.com/v1
   📝 Error: Connection error.
⏳ Retrying in 2.5s (attempt 1/3)...
⚠️  API call failed (attempt 2/3): APIConnectionError
   ...
🔁 Transient APIConnectionError on deepseek — rebuilt client, waiting 6s before one last primary attempt.
❌ API failed after 3 retries — Connection error.
```

## 根本原因

此问题由**两个独立原因叠加**导致：

| # | 层级 | 问题 | 影响 |
|---|------|------|------|
| 1 | 网络层 | WSL systemd-resolved DNS 失效 | 域名无法解析 → `Connection error` |
| 2 | 应用层 | Hermes DeepSeek 插件硬编码不稳定端点 | 即使 DNS 恢复，`/v1` 端点间歇性超时 |

两个问题单独出现时都可能导致连接失败，叠加后表现为 100% 不可用。

---

## 原因 1：WSL DNS 解析失效

### 诊断过程

```bash
# 步骤 1：从 Windows 宿主机测试 → 正常
curl -s -o /dev/null -w "HTTP: %{http_code}\n" --connect-timeout 10 https://api.deepseek.com/v1/models
# 输出: HTTP: 401（401 = 服务器响应了，只是没带 key）

# 步骤 2：从 WSL 内部测试 → 失败
wsl -d Ubuntu-24.04 -- bash -c "curl -v --connect-timeout 5 https://api.deepseek.com/v1/models"
# 输出: Could not resolve host: api.deepseek.com → curl: (6) 域名解析失败
```

### 定位根因

```bash
# 在 WSL 内执行
resolvectl status
```

关键输出：
```
Link 2 (eth0)
    Current Scopes: none          ← DNS 作用域为空！
         Protocols: -DefaultRoute -LLMNR -mDNS -DNSOverTLS DNSSEC=no/unsupported
```

`Current Scopes: none` 表示 systemd-resolved 没有为 eth0 网卡配置任何上游 DNS 服务器。这是 WSL2 + systemd 模式下的常见 bug —— systemd-resolved 无法从 Windows 宿主机获取 DNS 配置。

```bash
# 进一步验证
resolvectl query api.deepseek.com
# 输出: resolve call failed: No appropriate name servers or networks for name found
```

> 注意：此时 `ping 8.8.8.8` 是通的，说明网络路径正常，仅仅是 DNS 解析层故障。

### 修复

```bash
# 1. 删除 systemd-resolved stub 软链接
sudo rm /etc/resolv.conf

# 2. 写入静态 DNS 服务器
echo "nameserver 8.8.8.8" | sudo tee /etc/resolv.conf
echo "nameserver 114.114.114.114" | sudo tee -a /etc/resolv.conf
echo "nameserver 223.5.5.5" | sudo tee -a /etc/resolv.conf
```

> 说明：`/etc/wsl.conf` 中已有 `generateResolvConf=false`，WSL 不会自动覆盖修改后的文件。

### 验证

```bash
getent hosts api.deepseek.com
# 应返回 IP 地址和域名
curl -s -o /dev/null -w "HTTP: %{http_code}\n" --connect-timeout 10 https://api.deepseek.com/v1/models
# 应返回 HTTP: 401（而非 000）
```

---

## 原因 2：DeepSeek 插件硬编码了不稳定的 `/v1` 端点

### 诊断过程

DNS 修复后，测试两个 DeepSeek API 端点的可用性：

```bash
# 反复测试 3 轮，对比两个端点
for i in 1 2 3; do
  echo "=== /anthropic (第${i}次) ==="
  curl -s -w "HTTP: %{http_code} | Time: %{time_total}s\n" --connect-timeout 5 -m 8 \
    https://api.deepseek.com/anthropic/v1/messages \
    -H 'Content-Type: application/json' \
    -H "x-api-key: $DEEPSEEK_API_KEY" \
    -d '{"model":"deepseek-v4-pro","messages":[{"role":"user","content":"hi"}],"max_tokens":1}'

  echo "=== /v1 (第${i}次) ==="
  curl -s -w "HTTP: %{http_code} | Time: %{time_total}s\n" --connect-timeout 5 -m 8 \
    https://api.deepseek.com/v1/chat/completions \
    -H 'Content-Type: application/json' \
    -H "Authorization: Bearer $DEEPSEEK_API_KEY" \
    -d '{"model":"deepseek-chat","messages":[{"role":"user","content":"hi"}],"max_tokens":1}'
done
```

结果：

| 轮次 | `/anthropic` | `/v1` |
|------|-------------|-------|
| 1 | **200** (1.0s) | 超时 (5s) |
| 2 | **200** (1.2s) | 超时 (5s) |
| 3 | **200** (1.1s) | 200 (5.3s) |

**结论：`/anthropic` 端点每次都快速响应 (~1s)，`/v1` 大部分时间超时，极不稳定。**

### 为什么 Hermes 在用 `/v1` 而不是 `/anthropic`

Hermes 的 DeepSeek provider 插件位于：

```
~/.hermes/hermes-agent/plugins/model-providers/deepseek/__init__.py
```

原始代码：

```python
deepseek = ProviderProfile(
    name="deepseek",
    ...
    base_url="https://api.deepseek.com/v1",     # ← 硬编码了不稳定的 /v1
    # api_mode 未设置，默认 "chat_completions"  # ← 与 /anthropic 不兼容
)
```

即使 `~/.hermes/config.yaml` 中配置了：

```yaml
providers:
  deepseek:
    api: https://api.deepseek.com/anthropic
    transport: anthropic_messages
```

插件内的 `base_url` 没有被 config 覆盖，实际请求仍然发往 `https://api.deepseek.com/v1`。

### 修复

编辑 `~/.hermes/hermes-agent/plugins/model-providers/deepseek/__init__.py`，修改两个参数：

```python
deepseek = ProviderProfile(
    name="deepseek",
    aliases=("deepseek-chat",),
    env_vars=("DEEPSEEK_API_KEY",),
    display_name="DeepSeek",
    description="DeepSeek — Anthropic-compatible endpoint",
    signup_url="https://platform.deepseek.com/",
    fallback_models=(
        "deepseek-chat",
        "deepseek-reasoner",
    ),
    base_url="https://api.deepseek.com/anthropic",   # ← 改为可靠的 /anthropic
    api_mode="anthropic_messages",                    # ← 必须与端点格式匹配
)
```

修改后重启 Hermes 生效。

---

## 完整修复清单

| 步骤 | 操作 | 位置 |
|------|------|------|
| 1 | 修复 WSL DNS | `sudo rm /etc/resolv.conf` + 写入静态 DNS |
| 2 | 验证 DNS | `getent hosts api.deepseek.com` |
| 3 | 修改 provider 插件 | `base_url` → `/anthropic`，`api_mode` → `anthropic_messages` |
| 4 | 重启 Hermes | `pkill -f hermes && hermes` |

---

## 相关配置

修复后的关键配置文件：

**`~/.hermes/config.yaml`（provider 部分）：**
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

fallback_providers:
  - provider: deepseek
    model: deepseek-v4-flash
    api_mode: anthropic_messages
    base_url: https://api.deepseek.com/anthropic
```

**`~/.hermes/.env`：**
```bash
DEEPSEEK_API_KEY=sk-xxxxxxxx
```

---

## 为什么 `/v1` 和 `/anthropic` 稳定性不同

两个端点都指向 `api.deepseek.com` 的同一个 IP，但可能经过不同的后端网关/负载均衡：

- `/anthropic` 是 DeepSeek 为 Anthropic Messages API 兼容提供的端点，后端独立部署，线路优化较好
- `/v1` 是 DeepSeek 原生 API（OpenAI 兼容格式），从 WSL NAT 网络栈出站时可能触发某些 CDN/防火墙规则导致间歇性超时

如果你的网络环境中 `/v1` 更稳定，应反向操作：将 `base_url` 保持为 `https://api.deepseek.com/v1`，`api_mode` 保持为 `chat_completions`。

---

## 版本信息

- Hermes Agent: v0.13.0
- WSL: Ubuntu-24.04, systemd 模式
- 排障日期: 2026-05-10
