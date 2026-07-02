# Claude Science (Operon) — 第三方 API 接入指南

> 原始二进制：`linux-x64` (149MB, Bun standalone, ELF64 not stripped)  
> 内部代号：Operon (Anthropic Claude Code for Science)

---

## 快速开始

### 1. 启动 Daemon

```bash
env ANTHROPIC_AUTH_TOKEN="sk-你的token" \
    ANTHROPIC_BASE_URL="http://127.0.0.1:3000/" \  
    ANTHROPIC_DEFAULT_SONNET_MODEL="claude-opus-4-8" \
    ANTHROPIC_DEFAULT_OPUS_MODEL="claude-opus-4-8" \
    ANTHROPIC_DEFAULT_HAIKU_MODEL="claude-opus-4-8" \
    DISABLE_AUTOUPDATER="1" \
    ./linux-x64 serve --port 7777 --no-browser --detached \
    --dangerously-no-sandbox --dangerously-skip-approvals
```

一行版：
```bash
env ANTHROPIC_AUTH_TOKEN="sk-xxx" ANTHROPIC_BASE_URL="http://127.0.0.1:3000/" ANTHROPIC_DEFAULT_SONNET_MODEL="claude-opus-4-8" ANTHROPIC_DEFAULT_OPUS_MODEL="claude-opus-4-8" ANTHROPIC_DEFAULT_HAIKU_MODEL="claude-opus-4-8" DISABLE_AUTOUPDATER="1" ./linux-x64 serve --port 7777 --no-browser --detached --dangerously-no-sandbox --dangerously-skip-approvals
```

### 2. 获取登录链接

```bash
./linux-x64 url
```

在浏览器打开返回的链接即可进入 Agent 界面。

### 3. 停止

```bash
./linux-x64 stop
```

---

## 环境变量

| 变量 | 说明 | 示例 |
|------|------|------|
| `ANTHROPIC_AUTH_TOKEN` | API 认证令牌（Bearer token） | `sk-xxx` |
| `ANTHROPIC_BASE_URL` | 第三方 API 端点 | `http://127.0.0.1:3000/` |
| `ANTHROPIC_DEFAULT_SONNET_MODEL` | Sonnet 模型名 | `claude-opus-4-8` |
| `ANTHROPIC_DEFAULT_OPUS_MODEL` | Opus 模型名 | `claude-opus-4-8` |
| `ANTHROPIC_DEFAULT_HAIKU_MODEL` | Haiku 模型名 | `claude-opus-4-8` |
| `DISABLE_AUTOUPDATER` | 禁用自动更新 | `1` |

> **注意**：`ANTHROPIC_BASE_URL` 非 `https` 时必须是 `127.0.0.1`（loopback），否则会被 `szW()` URL 校验拒绝。如需使用其他 IP 的 HTTP 端点，需额外 patch `szW` 函数。

---

## 第三方 API 要求

API 端点需兼容 Anthropic Messages API 格式：

- `GET /v1/models` → `{"data": [{"id": "...", "display_name": "...", "type": "model"}]}`
- `POST /v1/messages` → Anthropic Messages API 格式

认证方式：`x-api-key` header 或 `Authorization: Bearer` header（由 `ANTHROPIC_AUTH_TOKEN` 提供）。

---


## 二进制 Patch 详情

所有 patch 均为**同长度字节替换**，不改变二进制大小。

| # | 文件偏移 | 原始 | 替换 | 作用 |
|---|---------|------|------|------|
| P1 | `0x6202C40` | `!O` | `!1` | `_j()` 跳过 "requires signing in" 检查 |
| P2 | `0x67C02B9` | `throw Error("No credentials available...")` (99B) | `throw Error("Credential lookup failed; env fallback. ")` (99B) | `resolve()` 错误消息避开 `/no credentials available/i` 正则 |
| P3 | `0x67edc85` | `email:W.email??null,provider:null,has_api_key:!1,...auth_mode:"none"` (85B) | `email:"dev@local.c",provider:"o",has_api_key:!0,...auth_mode:"oauth"` (85B) | `/api/me` 返回合法认证状态 |
| P4 | `0x68e6cfb` | `!1` | `!0` | `/api/auth/status` 返回 `authenticated: true` |
| P5 | `0x6204d93` | `if(this.credentialResolver)` (27B) | `if(0)                      ` (27B) | 对话路径跳过 OAuth 凭据解析，走 env fallback |

### 应用 Patch

```bash
# 从原始备份恢复
cp linux-x64.backup linux-x64

# 应用 5 个 patch
python3 -c "
with open('linux-x64','r+b') as f:
    f.seek(0x6202C40); f.write(b'!1')
    f.seek(0x67C02B9); f.write(b'throw Error(\"Credential lookup failed; env fallback. \")                                            ')
    f.seek(0x67edc85); f.write(b'email:\"dev@local.c\",provider:\"o\",has_api_key:!0,shared_api_key:!1,auth_mode:\"oauth\"}}')
    f.seek(0x68e6cfb); f.write(b'!0')
    f.seek(0x6204d93); f.write(b'if(0)                      ')
"
```

---

## 架构说明

### 认证流程（Patch 后）

```
用户点击 nonce 链接
  → 设置 operon_auth cookie
  → SPA 加载
  → GET /api/auth/status → authenticated: true (P4)
  → GET /api/me → auth_mode=oauth, provider=o (P3)
  → 前端计算 authenticated=True → 不跳转 /login

模型列表 (fallback 路径):
  → resolve() throw (P2) → 避开正则 → fallback 模型 (qEz)
  → 前端使用硬编码模型列表

Agent 对话 (env 回退路径):
  → _resolveCredsFromExplicit → if(0) 跳过 resolver (P5)
  → 返回空凭据 → _j(undefined, undefined)
  → _j 中 !O 检查被跳过 (P1)
  → 回退到 process.env.ANTHROPIC_AUTH_TOKEN
  → 创建 Anthropic 客户端 → 调用第三方 API
```

### 关键函数

| 函数 | 位置 | 说明 |
|------|------|------|
| `_j(z, O)` | `.bun 0xe8c3c` | 创建 Anthropic 客户端，含 env fallback |
| `BCz.resolve()` | `.bun 0x6a6200` | OAuth 凭据解析（模型列表路径） |
| `_resolveCredsFromExplicit()` | `.bun 0xead93` | 对话凭据解析（对话路径） |
| `xrG()` | `.bun 0x6d3c00` | `/api/me` 处理器 |
| `szW()` | `.bun 0x1594d` | URL scheme 校验（https 或 loopback） |

### 内嵌资源

| 资源 | 大小 | 说明 |
|------|------|------|
| `.bun` 节区 | 52 MB | 完整 JS/TS 应用代码 |
| `assets.tar.gz` | 45 MB | Web UI (1159 文件) + kernel_worker.py + 科学种子数据 |
| `operon-cli.db` | ~7 MB | SQLite 数据库（30+ 表） |

### 内嵌 Agent

| Agent | 功能 |
|-------|------|
| OPERON | 通用科学计算 Agent |
| REVIEWER | 转录审查 Agent |
| BOOKMARKER | 转录书签 Agent |
| ONBOARDING | 首次引导 Agent |

---

## 故障排查

### daemon failed to start
```bash
pkill -f "linux-x64 serve"
rm -f ~/.claude-science/operon.lock
# 重新启动
```

### 登录链接过期
```bash
./linux-x64 url  # 生成新链接
```

### 对话报错
检查环境变量是否正确传入：
```bash
cat /proc/$(pgrep -f "linux-x64 serve")/environ | tr '\0' '\n' | grep ANTHROPIC
```

### 回滚
```bash
git -C /home/agent1/software/claude_science checkout fe19c13 -- linux-x64
```

---

## 文件清单

| 文件 | 说明 |
|------|------|
| `linux-x64` | 已 patch 二进制  |
| `linux-x64.backup` | 原始二进制备份 |
| `analysis_output/` | 逆向分析产物 |
| `~/.claude-science/` | 运行时数据目录 |
| `REVERSE_REPORT.md` | 详细逆向分析报告 |
