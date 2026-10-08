# mist

A privacy passthrough relay for the Anthropic and OpenRouter APIs, on Cloudflare Workers.

Your requests leave from a clean, fixed edge egress — not from your device.

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/wusaki0723/mist)

---

## Why

Official OAuth login flows collect client-side device and environment signals. mist sits in between as a transparent relay: your client talks to mist, mist talks to `api.anthropic.com`. Upstream sees a uniform, minimal request shape from a consistent edge location — your device fingerprint and network environment stay out of the picture.

mist **stores nothing and inspects nothing** beyond what is required to normalize auth headers in flight. It is a pipe, not a product. The one exception: when the upstream returns an error or the proxy itself throws, mist writes a single structured line to Cloudflare Workers Logs (path, ray id, upstream status, a short excerpt of the upstream error) so failures can be diagnosed. Credentials, API keys, and request bodies are never logged.

## Routes

| Path | Upstream | Behavior |
|---|---|---|
| `/v1/*` | `api.anthropic.com` | Shaped to match the Claude Agent SDK. UA and `x-app` fill gaps only; beta flags merge on `/v1/messages*` alone; the Claude Code system prefix is added there when missing. Both OAuth setup-tokens and plain API keys work |
| `/or/*` | `openrouter.ai/api/*` | Pure passthrough. No disguise, body streamed byte-for-byte; mist only attaches your `OPENROUTER_API_KEY` |
| `/health` | — | Unauthenticated status check: which routes are configured, which auth mode is live |

## What it is

- **Pure passthrough** — streaming, tool use, vision: whatever the upstream supports, mist forwards untouched
- **Long-lived token, zero state** — one ~1-year token held in Cloudflare's secret store; no refresh flow, no KV, no database. Clients authenticate to mist with your own `PROXY_API_KEY`; the real token never leaves the edge
- **Agent SDK signature** — outgoing Anthropic requests are shaped to match the Claude Agent SDK, preserving full model compatibility (optional, can be disabled)
- **Header hygiene** — edge- and proxy-injected headers (`cf-*`, `x-real-ip`, `x-forwarded-*`, `true-client-ip`) are stripped before forwarding, so the upstream never sees your real client IP or hop chain
- **One-click deploy** — no resources to provision

## Recommended use

mist is meant for **Claude Code and official Claude SDKs only**, via their standard custom-endpoint settings (`ANTHROPIC_BASE_URL` / SDK `baseURL`). It is not a general-purpose API gateway and is shared at your own discretion within your own circle.

## ⚠️ Use at your own risk

mist is a personal networking tool. Whether routing your account's traffic through a relay complies with upstream terms of service is your own determination to make. Account restrictions are a possibility you accept by using this. Provided as-is, without warranty of any kind; the authors accept no liability for account actions or any other damages. If losing access to your account would hurt, don't use it.

## Setup

### 1. Get a long-lived token

Requires an active Claude subscription and Claude Code installed:

```bash
claude setup-token
```

Authorize in the browser; the terminal prints a token valid for ~1 year. **It is shown once — copy it immediately.**

### 2. Deploy

Click the button above, connect your GitHub account, and Cloudflare will fork and deploy the Worker for you.

Prefer the CLI?

```bash
git clone https://github.com/wusaki0723/mist
cd mist
npm install
npx wrangler login
npm run deploy
```

### 3. Add secrets

Cloudflare dashboard → **Workers & Pages → mist → Settings → Variables and Secrets → Add** (type: Secret):

| Secret | Value |
|---|---|
| `PROXY_API_KEY` | a long random string you invent — this is the key your clients will use |
| `CLAUDE_OAUTH_TOKEN` | the token from step 1 (preferred when set) |
| `ANTHROPIC_API_KEY` | optional: a regular `sk-ant-api03-...` key. Used when no OAuth token is set, and as an automatic one-time fallback when OAuth returns 401 |
| `OPENROUTER_API_KEY` | optional: your `sk-or-...` key, enables the `/or/*` passthrough |

Saving a secret triggers a new deployment automatically.

### 4. Verify

```bash
curl https://<your-worker>.<subdomain>.workers.dev/health
# → { "hasToken": true, "looksLikeSetupToken": true, ... }
```

### 5. Connect your client

**Claude Code:**

```bash
export ANTHROPIC_BASE_URL="https://<your-worker>.<subdomain>.workers.dev"
export ANTHROPIC_API_KEY="<PROXY_API_KEY>"
claude
```

**Official SDKs:** set base URL to your worker address (no `/v1` suffix), API key to your `PROXY_API_KEY`, and use official model ids.

**OpenRouter clients:** set the OpenRouter base URL to `https://<your-worker>.<subdomain>.workers.dev/or` and the API key to your `PROXY_API_KEY`. Every `/or/*` request is forwarded to `openrouter.ai/api/*` untouched.

**Hana:** add an Anthropic provider with base URL = worker address and key = `PROXY_API_KEY`; add an OpenRouter provider with base URL = worker address + `/or`, same key.

## Options

Request normalization (Anthropic path) is on by default. To disable it (model availability may be reduced):

```bash
npx wrangler secret put CLOAK   # enter: false
```

## When the token expires

Upstream starts returning 401 after ~1 year. Then:

```bash
claude setup-token                       # fresh token
npx wrangler secret put CLAUDE_OAUTH_TOKEN
```

No redeploy needed.

## Security notes

- `PROXY_API_KEY` is the only thing between the public internet and your account. Make it long and random.
- Never commit secrets; mist reads them from Cloudflare's secret store only.
- The `*.workers.dev` URL is publicly reachable. Rotate both secrets if you suspect leakage.

---

# 中文说明

Anthropic 与 OpenRouter API 的隐私透传中继，跑在 Cloudflare Workers 上。

你的请求从一个干净、固定的边缘出口发出，而不是从你的设备。

## 路径

| 路径 | 上游 | 行为 |
|---|---|---|
| `/v1/*` | `api.anthropic.com` | 出站形状对齐 Claude Agent SDK：UA 与 `x-app` 只补缺口；beta 标记只在 `/v1/messages*` 合并；Claude Code system 前缀缺了才补。OAuth setup-token 与普通 API key 都能用 |
| `/or/*` | `openrouter.ai/api/*` | 纯透传，不伪装，body 逐字节流式转发；mist 只负责注入 `OPENROUTER_API_KEY` |
| `/health` | — | 无需鉴权的自检：各路径配置与认证模式一目了然 |

## 为什么

官方 OAuth 登录流程会采集客户端的设备与环境信息。mist 夹在中间做透明中继：你的客户端连 mist，mist 连 `api.anthropic.com`。上游看到的只是一个来自固定边缘节点的、形态统一的干净请求——你的设备指纹和网络环境不会暴露。

mist **不存储、不窥探**任何请求内容，只在转发途中做必要的认证头归一化。它是一根管子，不是一个产品。唯一的例外：当上游返回错误或代理自身抛异常时，mist 会向 Cloudflare Workers Logs 写一条结构化日志（路径、ray id、上游状态码、错误摘要），方便排查问题。凭据、API key 和请求体永远不会被记录。

## 推荐使用

mist 仅推荐配合 **Claude Code 和官方 Claude SDK** 使用，走它们标准的自定义端点设置（`ANTHROPIC_BASE_URL` / SDK 的 `baseURL`）。它不是通用 API 网关，请在自己的小圈子里自行斟酌分享。

## ⚠️ 风险自担

mist 是个人网络工具。把账号流量经由中继转发是否符合上游服务条款，由你自己判断。使用即表示你接受账号可能被限制的风险。项目按原样提供，无任何担保；作者不对账号处置或其他损失负责。丢不起这个号，就别用。

## 上手

1. **拿 token**（需要有效订阅 + 本机 Claude Code）：`claude setup-token`，浏览器授权后终端打印一年期 token，**只显示一次，立刻复制**
2. **部署**：点上方按钮，或 `npm install && npx wrangler login && npm run deploy`
3. **加 secret**：CF 面板 → Workers & Pages → mist → Settings → Variables and Secrets，类型 Secret：
   - `PROXY_API_KEY`：自编长随机串（客户端用它认证）
   - `CLAUDE_OAUTH_TOKEN`：第 1 步那串（配了就优先用它）
   - `ANTHROPIC_API_KEY`：可选，普通 `sk-ant-api03-...` key。没配 OAuth token 时启用；OAuth 返回 401 时自动用它兜底重试一次
   - `OPENROUTER_API_KEY`：可选，你的 `sk-or-...`，配上即开启 `/or/*` 透传
4. **验证**：`curl https://<你的worker>.<subdomain>.workers.dev/health` 看各路径状态
5. **接客户端**：
   - Claude 系：base URL 填 Worker 地址（不带 `/v1`），API key 填 `PROXY_API_KEY`
   - OpenRouter 系：base URL 填 Worker 地址 + `/or`，key 同样填 `PROXY_API_KEY`
   - Hana：Anthropic provider 填 worker 地址，OpenRouter provider 填 worker 地址 + `/or`，key 都是 `PROXY_API_KEY`

## 可选旋钮

- `CLOAK=false`：关闭 Anthropic 路径的 system 前缀补全（可用模型范围可能缩小）
- `CLIENT_UA`：覆盖默认客户端 UA。内置默认是 `claude-cli/2.1.293 …`，版本老去时改它即可，不用动代码。`/health` 里能看到当前生效值

指纹优先级是硬规则：**客户端自己发的头原样透传，默认值只补缺口**。真 Agent SDK 调进来时形状端到端不变。

## License

[MIT](LICENSE)
