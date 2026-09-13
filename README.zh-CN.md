<div align="center">

# cnb2api

**将 CNB 云原生工作区内网 AI 转化为高可用生产级 API 网关**

*OpenAI + Anthropic 双协议栈 · Claude Code 零配置直连 · 0 依赖 · v5.2 自愈高可用 · 终端额度看板*

[![test](https://github.com/dengyie/cnb2api/actions/workflows/test.yml/badge.svg)](https://github.com/dengyie/cnb2api/actions/workflows/test.yml)
![node](https://img.shields.io/badge/node-%E2%89%A522-brightgreen)
![dependencies](https://img.shields.io/badge/dependencies-0-success)
[![license](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

[English](README.md) · 简体中文

</div>

> **一句话简介**：CNB 每月免费赠送 **500~1,166 AI Credits** 与 **1,600 核时**，但其 AI 端点严格受限于工作区内网与流水线临时 Token，且容器每天过夜回收。**cnb2api** 部署在工作区内，零外部依赖将其转换为标准公网 API，并通过独创的 **v5.2 双保险自愈架构** 实现永久固定域名访问与 7×24 小时高可用。

---

### 为什么选择 cnb2api？

| 维度 | cnb2api（本项目） | 社区逆向 NPC 方案 | 直接公网调用 |
|:---|:---|:---|:---|
| **调用途径** | **官方工作区内网端点** | 网页前端匿名游客 NPC 接口 | 官方公网端点 |
| **账号合规** | **100% 官方合规**，消耗自有正规配额 | 逆向抓包、违反 ToS、易封号 | 无法调用（网络策略拦截） |
| **网络可达性** | 内网反代 + 动态中继，**公网稳定访问** | 需维护无头浏览器/代理池抓 CSRF | ❌ 403 `Blocked by network policy` |
| **鉴权机制** | 流水线即时签发 `CNB_TOKEN`（零保管） | 抓取网页 cookie / session | ❌ 个人令牌 403（仅限流水线） |
| **协议支持** | **OpenAI + Anthropic 双协议栈** | 仅部分模拟 OpenAI 基础文本 | 无 |
| **Claude Code 直连** | **原生直连**（内置 11128 误报规避） | ❌ 不支持（缺 Tool Calling / 被拦截） | 无 |
| **原生 Tool Calling** | **完整透传**（流式增量合并 + 非流式聚合） | ❌ 不支持 | 无 |
| **外部运行依赖** | **0 依赖**（Node.js 22+ 原生标准库） | 需安装 Chromium / Puppeteer | 无 |
| **高可用机制** | **v5.2 双保险自愈**（内网 cron + 外部看门狗） | 页面一改版即报废 | 无 |
| **额度透明度** | **内置终端看板 CLI** + `/usage` 统计 | 黑盒无明细 | 仅控制台深处可看 |

---

### 核心特性

- ⚡ **双协议栈原生支持**：
  - **OpenAI 兼容**：`/v1/chat/completions` 与 `/v1/models`，支持流式 SSE 透传与忠实的非流式状态机聚合（完整回填 `usage`、真实 `credit` 扣费、`tool_calls` 与 `finish_reason`）。
  - **Anthropic 兼容**：`/v1/messages` 与 `/v1/messages/count_tokens`，**Claude Code CLI 零中间件直连**，完整支持思考链（thinking）、多轮工具调用与事件流。
  - 🛡️ **出站中性化规避**：自动等价重写 Claude Code 内部计费头与特征提示词短语，语义无损消除上游网关对 Claude Code 的 11128 误拦截。
- ♻️ **v5.2 零凭据 + 双保险自愈架构**：
  - **凭据零保管**：工作区内零静态长期 Token，定时自愈流水线每次使用平台即时签发、任务结束即销毁的临时 `CNB_TOKEN`。
  - **动态域名自动跟随**：工作区开机自动向中继注册新子域名（`/ops/register`），中继热重载 upstream，客户端固定域名与 Key 永久不变；实测自愈恢复仅需 **~2 分 13 秒**。
  - **防休眠看门狗**：外部看门狗（`cnb-watchdog.sh`）监控 CI 平台定时流水线休眠机制，连续失败时自动触发带冷却锁的 OpenAPI 应急拉起，23 秒恢复且真故障才报警。
- 🪶 **极简零依赖（Zero Dependencies）**：纯 Node.js 22+ 内置模块实现（原生 `fetch`、`ReadableStream`、`crypto.timingSafeEqual`、`node:test`），零 `npm install`，毫秒启动。
- 📊 **多维用量监控与独家额度看板**：
  - 终端一键查额度 `cnb2api-quota`：彩色 ANSI 进度条直观展示 AI Credits、开发核时、CI 核时及在途冻结额度，支持 `--json` 与 `--line`。
  - 内存级 `GET /usage` 统计端点，按 boot 隔离并支持 UTC+8 每日对账，完美适配自建看板与探针监控。

---

## 工作原理

```
客户端 (OpenAI 客户端 / Claude Code / API 网关)
  │  https://ai.example.com/v1 (固定域名，自建 Nginx 中继)
  ▼
https://<动态子域名>-9001.cnb.run (CNB 端口代理，每次开机通过 /ops/register 自动同步)
  │
  ▼
node src/server.mjs (工作区内网 9001 端口，cnb2api 反代)
  │  Bearer CNB_TOKEN (平台流水线自动签发，CNB 内网)
  ▼
https://api.cnb.cool/<org>/<repo>/-/ai/chat/completions (官方 AI 核心端点)
```

工作区被当作**牲口而非宠物（Cattle, not Pet）**：
1. **定时自愈**：平台内网 cron（`*/5 * * * *`）探活，发现工作区停止自动发起 `workspace/start`。
2. **开机自注册**：工作区启动后执行 `start.sh`，将新分配的子域名 POST 到中继的 `/ops/register`，Nginx 自动更新 upstream 并 reload。
3. **外部兜底**：VPS 外部看门狗（`cnb-watchdog.sh`）监控固定域名，防范 CI 平台长时间无提交导致的定时任务休眠。

---

## 快速开始

三步走，仅需配置一个 API Key：

1. **私有 Fork 本仓**：在 CNB 上将本仓库 Fork 为**私有仓库**。
2. **配置 API Key**：在仓库 `.cnb.yml` 的 `env:` 块填入自定义的 `PROXY_KEY`（如 `openssl rand -hex 24`）。
3. **启动云开发工作区**：点击 CNB 网页的「启动云原生开发」，构建日志打印出 `PROXY_URI=https://<子域名>-9001.cnb.run` 即可直接使用！

> 💡 需要永久固定域名？只需配置一个 VPS 运行简单的 Nginx 反代并开启 `/ops/register`，详见 [完整部署指南 (docs/SETUP.md)](docs/SETUP.md)。

---

## 客户端接入示例

### 1. Claude Code 直连（原生 Anthropic 协议）
```bash
ANTHROPIC_BASE_URL=https://ai.example.com \
ANTHROPIC_AUTH_TOKEN=your-proxy-key \
ANTHROPIC_MODEL=deepseek-v4-flash \
claude -p "你好，请介绍一下你自己"
```

### 2. OpenAI 客户端 / 聚合网关
将 BaseURL 指向 `https://ai.example.com/v1`，API Key 填入 `PROXY_KEY`：
- **聊天应用**：NextChat、LobeChat、Cherry Studio、Open WebUI
- **网关渠道**：New API、One API（渠道类型选择 OpenAI，模型填 `deepseek-v4-flash`）
- **代码插件**：Cursor、Continue、Codex CLI

### 3. curl 调用验证
```bash
curl https://ai.example.com/v1/chat/completions \
  -H "Authorization: Bearer your-proxy-key" \
  -H "Content-Type: application/json" \
  -d '{"model":"deepseek-v4-flash","messages":[{"role":"user","content":"hi"}],"stream":false}'
```

---

## 额度与成本测算

基于生产环境长期运行实测数据：

| 资源配额 | 每月免费总量 | 消耗与折算说明 |
|:---|:---|:---|
| **AI Credits** | **500 ~ 1,166** | 实测未命中缓存约 **2.3 万 tokens/credit**；命中 Prompt Cache 降至原价 1/30。90% 缓存命中率下，1,166 credits ≈ **2 亿 tokens/月**。 |
| **算力核时** | **1,600 核时** | `runner.cpus: 2` 工作区满载运行 30 天 = **1,440 核时**（在免费池内）。无需刻意关机。 |

---

## 额度看板 (CLI)

在终端一键直连官方 Charge 计费接口，即使反代未开机也能查询：

```bash
npm run quota                 # 或: npx cnb2api-quota
```

```text
  ◆ CNB quota  your-org

  Credits  ███████▋───────────────────  32%   320.0 / 1,000.0 cr
  Dev      ██████▏─────────────────────  25%   406.4 / 1,600.0 core-h
  CI       █▋──────────────────────────   8%   13.0 / 160.0 core-h

  in-flight (not yet settled): 12.0 cr, 0.8 core-h
  remaining credits: 680.0 cr   as of 2026-01-15 08:30:00 UTC
```

- `cnb2api-quota --line`：单行紧凑模式，适合嵌入状态栏或 Shell 提示符。
- `cnb2api-quota --json`：标准 JSON 输出，便于脚本采集自动化。

---

## 配置项参考

| 环境变量 | 默认值 | 说明 |
|:---|:---|:---|
| `PROXY_KEY` | — (必填) | 客户端访问反代网关的 Bearer Token / x-api-key。 |
| `CNB_TOKEN` | — (必填) | CNB 内网端点鉴权令牌，流水线环境自动注入。 |
| `CNB_REPO_SLUG` | (内置) | 当前仓库 `org/repo`，流水线内置自动填充。 |
| `PROXY_MODELS` | `deepseek-v4-flash,glm-5.3-flash,kimi-k3` | `/v1/models` 暴露的模型列表。 |
| `PROXY_PORT` | `9001` | 服务监听端口。 |
| `PROXY_UPSTREAM_TIMEOUT_MS` | `15000` | 上游连接首字节超时 (ms)。 |
| `PROXY_IDLE_TIMEOUT_MS` | `300000` | 流式事件空闲超时看门狗 (ms)。 |
| `REGISTER_URL` | — (可选) | 中继自注册 `/ops/register` 地址。 |
| `REG_TOKEN` | — (可选) | 中继自注册鉴权密钥。 |

---

## 常见问题 (FAQ)

**Q: 既然上游是 DeepSeek，为什么模型列表有 glm 和 kimi？**  
A: CNB 官方网关目前在底层统一将请求路由至 `deepseek-v4-flash`。暴露多个模型 ID 仅为了兼容各类客户端的默认预设，后续上游开放多模型时反代无需修改即可支持。

**Q: 原生函数/工具调用（Tool Calls）支持怎么样？**  
A: 完整原生支持。流式模式下支持 `input_json_delta` 增量合并，非流式模式下忠实聚合出标准的 `tool_calls` 结构。

**Q: 是否违反 CNB 用户条款？**  
A: 本项目完全运行在你的合法工作区中，调用官方公开文档中的正规端点，使用平台签发的流水线 Token，扣除的是你组织的自有配额。不逆向、不破解、不涉及灰产爬虫。

---

## 许可

MIT © [dengyie](https://github.com/dengyie/cnb2api)
