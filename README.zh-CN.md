<div align="center">

# cnb2api

**将 CNB 云原生工作区内网 AI 转化为高可用生产级 API 网关**

*原生支持 OpenAI + Anthropic 双协议栈 · Claude Code 零配置直连 · 0 依赖 · v5.2 双保险自愈高可用 · 终端额度看板*

[![test](https://github.com/dengyie/cnb2api/actions/workflows/test.yml/badge.svg)](https://github.com/dengyie/cnb2api/actions/workflows/test.yml)
![node](https://img.shields.io/badge/node-%E2%89%A522-brightgreen)
![dependencies](https://img.shields.io/badge/dependencies-0-success)
[![license](https://img.shields.io/badge/license-MIT-blue)](LICENSE)

[English](README.md) · 简体中文

</div>

CNB（云原生构建平台）为每个实名认证组织每月免费提供丰厚的开发者资源：**500 ~ 1,166 AI Credits**（搭配 Prompt Cache 相当于最高 **~2 亿 tokens** 可用量）与 **1,600 核时** 算力。然而，官方内置的 AI 对话补全端点存在严格的网络与环境限制：

1. 🔒 **内网物理隔离与流水线鉴权**：端点严格限定在 CNB 工作区内网（公网访问直接被网络策略拦截报 403 `Blocked by network policy`）；且鉴权必须使用流水线即时签发的临时 `CNB_TOKEN`（个人访问令牌报 403 `OpenAPI only allowed in pipeline`）。外部 IDE、Cursor、Claude Code 或三方 API 网关无法直连。
2. 🌊 **强制流式输出与模型路由**：官方 API 强制要求 `stream: true`（非流式请求直接报错 11101），且所有模型（如 deepseek-v4-flash、glm-5.3-flash 等）在底层网关经历统一路由与映射，需要健壮的协议转换与状态聚合。
3. ⏳ **易逝性云工作区生命周期（Cattle, not Pet）**：工作区无连接 10 分钟自动关机、单次运行最长 18 小时、凌晨 4-6 点强制「环境不过夜」回收；每次重新启动分配的端口代理子域名（`{subdomain}-9001.cnb.run`）均随机变化。
4. ⚠️ **社区逆向方案的脆弱与合规风险**：社区同类项目多采用抓取前端匿名 NPC 聊天接口、无头 Chromium 会话池或逆向 CSRF 的方案。不仅缺乏原生 Tool Calling 支持，更易因平台改版/风控而彻底失效。

---

**cnb2api** 为此而生：它是一个专为 CNB 工作区打造的**极简、零依赖、100% 官方合规的高可用反向代理网关**。直接运行在工作区内网，不仅将官方内网 AI 端点无缝映射为标准稳定的公网接口，更独创 **v5.2 零长效凭据 + 双保险自愈架构**，在易逝的临时容器之上提供 7×24 小时高可用服务。

### 核心特性

- 🔓 **100% 官方合规正道**：只调用官方文档化的工作区 AI 端点，完全依靠平台原生流水线临时凭据。不逆向、不爬虫、不碰前端 NPC 会话池，原生支持 Function / Tool Calls。
- ⚡ **OpenAI + Anthropic 双协议栈**：
  - **OpenAI 兼容**：标准 `/v1/chat/completions` 与 `/v1/models`，支持流式 SSE 透传与忠实的非流式状态机聚合（完整回填 `usage`、真实 `credit` 扣费、`tool_calls` 与 `finish_reason`）。
  - **Anthropic 原生兼容**：原生 `/v1/messages` 与 `/v1/messages/count_tokens`，**Claude Code CLI 零中间件直连**（设置 `ANTHROPIC_BASE_URL` 与 `ANTHROPIC_AUTH_TOKEN` 即用），支持流式事件序列、thinking 思考链与双向工具调用映射。
  - 🛡️ **独家出站中性化规避（Upstream Neutralizer）**：自动等价重写 Claude Code 内部计费头与特征提示词短语，语义无损放行，彻底消除上游网关对 Claude Code 的 11128 误报拦截。
- ♻️ **v5.2 零凭据泄露 + 双保险自愈高可用**：
  - **凭据零保管**：工作区无任何长期静态 Token。定时自愈流水线每次使用平台现签现销的临时凭据，过期风险彻底归零。
  - **动态域名自动跟随**：工作区开机通过 `start.sh` 自动向中继注册新子域名（`/ops/register`），中继自动热重载 upstream，客户端固定域名与 Key 永久不变。实测完整自愈恢复仅需 **~2 分 13 秒**。
  - **v5.2 静默自愈双保险**：针对云平台连续多日无 commit 自动休眠 crontab 的平台机制，外部看门狗（`cnb-watchdog.sh`）在检测到异常时静默触发带 30 分钟冷却锁的 OpenAPI 应急拉起，23 秒恢复且真故障才告警，告警噪音降为零。
- 🪶 **真正的零外部依赖（Zero Dependencies）**：100% 基于 Node.js 22+ 内置模块实现（原生 `fetch`、`ReadableStream`、`crypto.timingSafeEqual`、`node:test`），零 `npm install`，毫秒级冷启动，内存占用极低。
- 📊 **全链路用量监控与独家额度看板**：
  - 终端一键查额度 `cnb2api-quota`：红黄绿 ANSI 进度条可视化展示 AI credits、开发核时、CI 核时及在途冻结额度，支持 `--json` 与 `--line`，不启动反代也能查。
  - 内存级 `GET /usage` 统计端点，按 boot 换代隔离并支持 UTC+8 每日对账，完美对接自建看板与探针监控。

---

### 技术方案横向对比

| 维度 | cnb2api（本项目） | 社区逆向 NPC 接口方案 | 直接外部调用 |
|:---|:---|:---|:---|
| **调用途径** | **官方工作区内网 AI 端点** | 网页前端匿名游客 NPC 聊天接口 | 官方 AI 端点（公网） |
| **合规与账号** | **100% 官方合规**，消耗自己组织正规配额 | 灰产逆向、违反 ToS、易被平台风控 | 无法调用（网络策略拦截） |
| **网络可达性** | 工作区反代 + 动态中继，**公网稳定访问** | 需维护无头浏览器/代理池抓 CSRF | ❌ **403 Blocked by network policy** |
| **鉴权机制** | 流水线即时签发 `CNB_TOKEN`（零保管） | 抓取网页 cookie / session | ❌ 个人令牌 403（仅允许流水线） |
| **协议支持** | **OpenAI + Anthropic 双协议栈** | 仅部分模拟 OpenAI 基础文本 | 无 |
| **Claude Code 直连** | **原生直连**（内置 11128 误报规避） | ❌ 不支持（缺少 tool calling / 被拦截） | 无 |
| **Tool Calling / 函数调用** | **原生完整支持**（流式增量合并） | ❌ 不支持（前端接口无 tools 能力） | 无 |
| **外部运行依赖** | **0 依赖**（Node.js 22+ 内置，秒启） | 依赖 Chromium、Puppeteer 等重型依赖 | 无 |
| **高可用机制** | **v5.2 双保险自愈**（内网 cron + 外部看门狗） | 单点容易随前端改版报废 | 无 |
| **额度透明度** | **内置终端看板 CLI** + `/usage` 统计 | 黑盒无账单，不可视 | 只能进网页控制台深处查看 |

## 工作原理

```
client ──https://ai.example.com/v1──▶ 固定域名(自建 relay / nginx)
                                        │  /ops/register 自动改写 upstream map
                                        ▼
                          https://<子域名>-9001.cnb.run   (CNB 端口代理)
                                        │
                                        ▼
                          node src/server.mjs   (本代理,跑在工作区里)
                                        │  Bearer CNB_TOKEN,仅 CNB 内网可达
                                        ▼
       https://api.cnb.cool/<org>/<repo>/-/ai/chat/completions   (CNB AI 端点)
```

`ai.example.com`、`<org>/<repo>`、端口 `9001` 都是**占位符**——换成你自己的。
固定域名是可选的;不用的话,直接用构建日志里那个
`https://<子域名>-9001.cnb.run/v1`。

### 在一台每天都死的机器上保持在线

工作区被当作**牲口,而非宠物**(cattle, not a pet)。一条 cron 流水线加一个
小中继,把一台每晚被回收的机器,变成一个像高可用 VPS 一样的端点:

```
每 5 分钟(cron 流水线)               每次开机                       你的中继
┌─────────────────────────────┐   ┌──────────────────────┐   ┌─────────────────────┐
│ 工作区还活着吗?             │   │ start.sh 运行 →      │   │ nginx upstream map  │
│  活着 + 域名正常 → 不动     │──▶│ POST /ops/register   │──▶│ 改指到最新的        │
│  域名死了       → 重注册    │   │ (当前子域名)         │   │ 子域名              │
│  没活着         → 拉起      │   └──────────────────────┘   └─────────────────────┘
└─────────────────────────────┘
```

- **自愈**——cron 拉起死掉的工作区;固定域名两次健康检查失败触发重注册。
  全程无人干预。
- **地址固定、后端可动**——客户端永远只看到一个 URL;工作区回收后,中继几
  分钟内改指到新子域名。实测一次真实回收演练,完整恢复(检出 → 新工作区 →
  重注册)耗时 **~2 分 13 秒**。
- **磁盘上不留长效令牌**——被回收的工作区什么都不带,下次开机按设计重新铸造
  一次性 `CNB_TOKEN`。你的 `PROXY_KEY` 和 `REG_TOKEN` 放在私有密钥仓,构建时
  才注入。
- **双保险看门狗兜底(可选)**——若云平台因仓库多日无提交而暂停定时流水线(休眠机制),
  中继 VPS 上的 `deploy/cnb-watchdog.sh` 可在连续失败后自动通过备用令牌调 API 拉起工作区,
  彻底消除平台定时任务休眠带来的单点风险。

这是**服务级**高可用,不是实例级:稳定 URL 与可用代理自动扛过回收。代价是
每次恢复的几分钟中断——对一个个人网关来说,零额外基础设施成本下这个取舍很
难被超越。设计细节见 [docs/DESIGN.md](docs/DESIGN.md)。

## 功能

- **OpenAI 兼容**——`/v1/chat/completions`(流式 SSE + 非流式)、`/v1/models`、
  `/health`。原生**函数/工具调用**原样透传。
- **Anthropic 兼容**——`/v1/messages`(含 `/v1/messages/count_tokens`)完整实现
  Anthropic Messages 协议,**Claude Code 直连**:`ANTHROPIC_BASE_URL` 一设就能用。
  请求/响应翻译覆盖 system 提示(顶级与 messages 内)、text/image/thinking 块、
  `tool_use`/`tool_result`,以及完整流式事件序列
  (`message_start → content_block_* → message_delta → message_stop`)。
- **忠实的非流式聚合**——把 SSE 流里的 `content`、增量 `tool_calls`、
  `reasoning_content`、`usage`、`finish_reason` 重组成一个完整的
  `chat.completion` 对象。
- **生产级加固转发**——连接超时、流级空闲看门狗、背压处理、双向取消(客户端
  断开会中止上游,不再为已放弃的请求烧额度)。
- **时序安全的 key 鉴权**——配滑动窗口失败限流(滥用回 429)。`Authorization:
  Bearer` 与 Anthropic 的 `x-api-key` 两种头等价。
- **用量端点**——鉴权 `GET /usage` 返回按 boot 的 token 累计
  (`prompt`/`completion`/`requests`/`errors`),方便外部监控对接。
- **额度看板 CLI**——`cnb2api-quota` 直连 CNB charge 接口,见[下文](#额度看板cli)。

## 快速开始 —— 部署到 CNB

三步,只填一个值。**私有 fork** 本仓,把自己的 key 填进 `.cnb.yml` 的 `env:`
块(私有仓里内联是安全的,可用 `openssl rand -hex 24` 生成),再启动云开发
工作区。构建日志会打印端点(`PROXY_URI=https://<子域名>-9001.cnb.run`)——
它加上你的 key 就是一个能用的 OpenAI base URL。其余零改动:keepalive 读的
是内置变量 `CNB_REPO_SLUG`,无需任何按用户配置。这条路径已在真实账号上端到端
走通:首个 cron 周期日志打出 `list http=200`,外部 curl 拿到 200 的对话补全。
完整流程(含团队级的密钥仓方案、可选固定域名中继)见 [docs/SETUP.md](docs/SETUP.md)。

## 你实际能得到什么

这套方案背后有两份相互独立的免费额度:**AI credits** 付推理费,**核时** 付
反代算力费。以下数字全部来自我们自己长期在线的部署——是实测,不是营销。

| 额度 | 每月免费配额 | 用来付什么 |
|---|---|---|
| **AI credits** | 基础 **500**,完成 *hello-cnb* 闯关后达 **1,166** | 每一次 AI 请求——每个响应的 `usage` 都精确上报本次消耗的 `credit` |
| **算力核时** | **1,600 核时**(dev + CI 共享池) | 跑反代的工作区,外加保活流水线 |

> 500 基础额度随实名组织发放;完成 CNB 官方 *hello-cnb* 闯关(「天才程序员」
> 徽章)会追加每月浮动奖励(我们账号合计 1,166/月)。你自己的总额以账号实际
> 状态为准。

**Credits → tokens(实测)。** 上游在每个响应的 `usage` 里直接返回 `credit`
字段,消耗是精确值。对 `deepseek-v4-flash` 的实测:

- 全新(未缓存)约 9,000 tokens 的请求花费约 **0.39 credit**——约
  **23,000 tokens / credit**,多次一致。
- 相同 prompt 命中 CNB 的 prompt 缓存后降到 **~0.01 credit**——缓存部分约为
  原价的 **1/30**。

所以综合单价取决于你的缓存命中率。按 **90% 命中率**(固定 system prompt、
Agent 循环反复读同一段上下文的场景)算,平均成本是
`10%×1 + 90%×(1/30) ≈ 13%` 的原价——约 **17.7 万 tokens/credit**,1,166
credits 约合 **2 亿 tokens/月**。别把任何单一数字当承诺:用下方的
[额度 CLI](#额度看板cli) 盯你自己的真实消耗。

**核时 → 在线时长。** `runner.cpus: 2` 的工作区每天烧 **48 核时**;整月 30 天
不间断 = **1,440 核时**,在 **1,600** 池子之内——不需要为省核时而关停反代。
5 分钟一次的保活单月只多花几个核时。CNB 单次会话上限 18 小时,运行超 8 小时
的工作区在 04:00–06:00(UTC+8)窗口会被回收,保活循环负责立刻拉回来。

## 模型

`/v1/models` 展示的就是你在 `PROXY_MODELS` 里列的清单。我们账号上,CNB 网关
当前暴露三个 id——而且当前都路由到同一个上游模型:

| 模型 id | 说明 |
|---|---|
| `deepseek-v4-flash` | 当前实际应答的模型。 |
| `glm-5.3-flash` | 可作为 id 调用,但被路由到 `deepseek-v4-flash`(响应的 `model` 字段可印证)。 |
| `kimi-k3` | 同上——当前同样路由到 `deepseek-v4-flash`。 |

额外的名字只是为了客户端兼容。`PROXY_MODELS` 请按你自己账号实际暴露的清单来
设。流式与非流式请求、完整 `usage` 聚合(含 `credit`)、原生 `tools` 调用均已
对线上端点实测验证。上下文窗口由上游决定且官方未文档化,我们不引用无法核实的
数字。

## 到处都能用

端点说的是标准 OpenAI chat completions,凡是支持自定义 base URL 的都能直接
用。把 base URL 指到你的固定域名(`https://ai.example.com/v1`),API key 填你
的 `PROXY_KEY`:

- **聊天客户端**——LobeChat、Cherry Studio、Open WebUI、NextChat……
- **编码 Agent / SDK**——Codex CLI、官方 `openai` SDK,或任何 OpenAI 兼容工具链。
- **Claude Code**——原生 Anthropic 协议,无需任何 shim:

  ```bash
  ANTHROPIC_BASE_URL=https://ai.example.com \
  ANTHROPIC_AUTH_TOKEN=<你的 PROXY_KEY> \
  ANTHROPIC_MODEL=deepseek-v4-flash \
  claude -p "hello"
  ```
- **curl**——见[本地开发](#本地开发)。

## 额度看板(CLI)

CNB 的额度和核时只在网页控制台深处能看。`cnb2api-quota` 把它变成终端里的一条
命令:

```bash
npm run quota                 # 或: npx cnb2api-quota
```

```
  ◆ CNB quota  your-org

  Credits  ███████▋───────────────────  32%   320.0 / 1,000.0 cr
  Dev      ██████▏─────────────────────  25%   406.4 / 1,600.0 core-h
  CI       █▋──────────────────────────   8%   13.0 / 160.0 core-h

  in-flight (not yet settled): 12.0 cr, 0.8 core-h

  remaining credits: 680.0 cr   as of 2026-01-15 08:30:00 UTC
```

红黄绿进度条(随消耗从绿到红)、千分位、以及已预留但尚未结算的 in-flight
额度。另有两种输出模式:

```bash
cnb2api-quota --json          # 给脚本用的规整快照
cnb2api-quota --line          # 状态栏 / shell 提示符的单行
```

它直连 CNB charge 接口(`/-/charge/quota` + `/-/charge/volume`),所以**反代
工作区没开机也能查**,且任何能看到 org 账单的令牌都行——不需要流水线令牌的
特殊权限。org 取自 `CNB_REPO_SLUG`,或用 `--org <org>` / `QUOTA_ORG` 覆盖。

## 本地开发

跑测试(mock 上游,不打真实 API):

```bash
node --test
```

用测试钩子把上游指到任意 OpenAI 风格服务,单机跑反代:

```bash
PROXY_KEY=my-secret \
CNB_TOKEN=dummy \
CNB_REPO_SLUG=your-org/ai-proxy \
UPSTREAM_OVERRIDE=http://127.0.0.1:8080 \
node src/server.mjs
```

```bash
curl http://127.0.0.1:9001/v1/chat/completions \
  -H "Authorization: Bearer my-secret" -H "Content-Type: application/json" \
  -d '{"model":"deepseek-v4-flash","messages":[{"role":"user","content":"hi"}],"stream":false}'
```

## 配置

| 环境变量 | 默认值 | 用途 |
|-----|---------|---------|
| `PROXY_KEY` | —(必填) | 客户端必须携带的 Bearer key。无默认值;缺失拒绝启动。 |
| `CNB_TOKEN` | —(必填) | 上游令牌;由 CNB 流水线阶段自动注入。 |
| `CNB_REPO_SLUG` | (内置) | 拼上游 URL 用的 `org/repo`。CNB 内置变量,所有流水线自动填充。 |
| `PROXY_MODELS` | `deepseek-v4-flash,glm-5.3-flash,kimi-k3` | `/v1/models` 展示的模型 id 列表。 |
| `PROXY_PORT` | `9001` | 监听端口。 |
| `PROXY_UPSTREAM_TIMEOUT_MS` | `15000` | 上游连接 / 首字节超时。 |
| `PROXY_IDLE_TIMEOUT_MS` | `300000` | 流级空闲看门狗。 |
| `REGISTER_URL` | —(可选) | 自注册的 relay `/ops/register` 地址。 |
| `REG_TOKEN` | —(可选) | 自注册共享密钥。 |
| `QUOTA_ORG` | —(可选) | 额度 CLI 查询的 org,与 `CNB_REPO_SLUG` 不同时使用。 |
| `UPSTREAM_OVERRIDE` | — | 仅测试:把上游指到本地 mock。 |

## FAQ

**和 GitHub 上那些匿名 CNB 代理有什么区别?**
根本区别。那些项目包装的是 CNB 给匿名网页访客用的前端 NPC 聊天接口——抓 CSRF
令牌、轮转会话池、网站一改版就得重新逆向协议。脆弱、无账号,也明显不是平台
本意。cnb2api 走的是**官方工作区 AI 端点**:文档化路径、你自己 org 的额度、
完整 `tools` 支持、用量记录在你自己账号上。花的是你自己的额度,而不是平台的
耐心——UI 改版也照样能用。

**原生工具调用能用吗?**
能。请求走官方端点、带你的流水线令牌,所以 `tools` / `tool_calls` 原样透传
——不需要任何提示词注入的绕行手段。

**实现了哪些 API 端点?**
`/v1/chat/completions`(SSE 流式 + 非流式)、`/v1/messages`(含
`count_tokens`)、`/v1/models`、`/usage`、`/health`。没有
embeddings/audio/files——上游本身也不提供。

**支持 Anthropic 格式的客户端吗?**
支持——`/v1/messages` 端到端实现 Anthropic Messages 协议(请求翻译、流式事件
序列、`count_tokens` 估算、Anthropic 风格错误 envelope)。Claude Code 设
`ANTHROPIC_BASE_URL` 即可直连。两个模型相关的边界:`thinking` 块入站丢弃、
出站经上游 `reasoning_content` 透传回传;`count_tokens` 是字符数/4 的估算——
做上下文百分比够用,不是精确值。协议适配层以生产环境先行验证,本仓跟随同步。

**跑起来要花什么成本?**
代码是 MIT 免费的。你花的是 CNB 额度:每次请求的 AI credits,加上保活撑开
工作区期间的核时(2 核约 48 核时/天——预算算法见上文)。

**这和 CNB 官方有关联吗?**
没有。独立的个人自用项目。请遵守平台服务条款。

## 许可

MIT——见 [LICENSE](LICENSE)。

> 与 CNB 无隶属关系。这是一个独立的、个人自用的兼容 shim。请遵守 AI 服务商
> 与平台的服务条款。

如果 cnb2api 帮你省下了一笔付费 API 订阅,欢迎点个 ⭐——能帮到更多 CNB 用户
发现它。
