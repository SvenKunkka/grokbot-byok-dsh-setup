<!-- SVENKUNKKA-README-NAV:BEGIN -->
[English overview](#project-overview-en) · [中文项目说明](#project-overview-zh) · [Maintainer on X / 维护者 X](https://x.com/Svenu1s5)
<!-- SVENKUNKKA-README-NAV:END -->

# grokbot-byok-dsh-setup

Bring-your-own-key (BYOK) setup for [DeepSeekBot](https://github.com/BotHarness/DeepSeekBot) on [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness): route Bot model traffic to OpenAI, Anthropic, OpenRouter or any OpenAI-compatible gateway instead of a single vendor account.

This repo is a **configuration template and field guide**, not a fork. It explains why the original Grok Bot client cannot take a third-party key, and gives a drop-in `cordis.patch.yml` fragment that makes DSB bots use your own keys.

<a id="project-overview-en"></a>
## English overview

### The problem

Two complaints show up together:

1. **Quota burns too fast** — a fixed account allowance disappears quickly once several bots run in groups on schedules.
2. **No third-party API key support** — you want to pay per token on your own provider, and cannot.

These are the same problem. A client that hard-codes one vendor's backend has no place to put a key.

### Why the original client can't be patched

Inspecting the shipped `app.asar` of the Grok Bot desktop client shows there is **no BYOK seam to patch**:

| Check | Result |
|---|---|
| `openai` literal in the main bundle | 0 occurrences |
| `anthropic` literal | 0 occurrences |
| `chat/completions` | 0 occurrences |
| Model request egress | vendor-operated backend only |
| Auth | account token, not a user-supplied key |
| Quota | account credit balance |

A patch would mean writing the entire multi-provider layer from scratch. Swapping the host is the smaller change — and that is what DSB already is.

### Why DSH works

DeepSeek Harness ships a multi-provider model layer. The `dsh-llm-pi-ai` adapter routes requests to:

- **OpenAI** (`api: openai-responses`)
- **Anthropic** (`api: anthropic-messages`)
- **Any OpenAI-compatible gateway** (`api: openai-completions` + your `baseURL`)

Keys are referenced by **environment-variable name** (`apiKeyEnv`), resolved through the credential store at request time. The config file never contains a secret.

DSB reads whatever provider you configure in DSH, so BYOK works without touching DSB's code. DSB additionally supports a **per-bot model plan**, which is the real answer to the quota complaint.

### Install

```bash
# 1. Install DSB into your DSH profile
cd ~/.dsh/profiles/<profile>
pnpm add deepseekbot@1.4.1

# 2. Enable it: add "deepseekbot" to dsh.profile.bundles in package.json

# 3. Add provider routes (see examples/cordis.patch.example.yml)

# 4. Verify the composed config before restarting
dsh --profile <profile> --dump-config | grep -A20 llm-pi-ai
```

Then open **Settings → Models**, click each provider row, and paste the key. The UI is write-only: saved credentials are never read back into the field. Use **Discover models** to pull the model list.

### Cut token spend

| Lever | Effect |
|---|---|
| **Per-bot model plan** | Cheap model for chat, expensive model only for code/research. Biggest win. |
| `/compact` + idle compaction | Long transcripts resend full history every turn. |
| Turn off non-essential review models | Each review is an extra model call. |
| Restrain schedule frequency | Every trigger is a full conversation. |
| Fewer bots in groups | Multiple bots replying = multiple calls. `@` only who should answer. |
| Context caching | Some providers cache prefixes at a large discount. |

### Rollback

Back up the profile directory before editing, then restore and remove the dependency:

```bash
cp -R ~/.dsh/profiles/<profile> ~/.dsh/profiles/<profile>.bak
# ...edit...
cp ~/.dsh/profiles/<profile>.bak/{cordis.patch.yml,package.json} ~/.dsh/profiles/<profile>/
cd ~/.dsh/profiles/<profile> && pnpm remove deepseekbot
```

### Never commit

- Real API keys (they belong in the credential store, not YAML)
- Internal gateway hostnames or IPs
- `.credentials.yaml` or any credential snapshot

<a id="project-overview-zh"></a>
## 中文项目说明

### 解决什么问题

你说的两件事其实是同一件事：

1. **额度用得太快** —— 账号给的固定额度，几个 Bot 在群里跑、又挂着定时任务，很快就见底。
2. **不支持第三方 API Key** —— 想自己按量付费走别的服务商，做不到。

一个把后端写死成单一厂商的客户端，没有地方放别人的 Key。

### 为什么原版改不了

拆开 Grok Bot 桌面客户端的 `app.asar` 可以看到，**根本没有可打的 BYOK 接缝**：

| 检查项 | 结果 |
|---|---|
| 主 bundle 里 `openai` 字面量 | 0 次 |
| `anthropic` 字面量 | 0 次 |
| `chat/completions` | 0 次 |
| 模型请求出口 | 仅厂商自有后端 |
| 鉴权方式 | 账号 token，不是用户提供的 Key |
| 额度 | 账号授信余额 |

要打补丁等于从零写一整套多服务商层。**换宿主**是改动更小的路子——而 DSB 本来就是这么做的。

### 为什么 DSH 可以

DeepSeek Harness 自带多服务商模型层，`dsh-llm-pi-ai` 适配器可以把请求路由到：

- **OpenAI**（`api: openai-responses`）
- **Anthropic**（`api: anthropic-messages`）
- **任意 OpenAI 兼容网关**（`api: openai-completions` + 你自己的 `baseURL`）

密钥以**环境变量名**引用（`apiKeyEnv`），请求时经凭据存储解析。**配置文件里永远没有密钥明文**。

DSB 读的是你在 DSH 里配的 provider，所以不用改 DSB 一行代码就能用上自己的 Key。DSB 另外支持**按 Bot 分配模型**，这才是"额度省"的正解。

### 安装

```bash
# 1. 把 DSB 装进你的 DSH profile
cd ~/.dsh/profiles/<profile>
pnpm add deepseekbot@1.4.1

# 2. 启用：把 "deepseekbot" 加进 package.json 的 dsh.profile.bundles

# 3. 加 provider 路由（见 examples/cordis.patch.example.yml）

# 4. 重启前先验证组装结果
dsh --profile <profile> --dump-config | grep -A20 llm-pi-ai
```

然后打开 **设置 → Models**，点各 provider 行填入 Key。界面是只写的：已保存的凭据不会被读回输入框。用 **发现模型** 拉取模型列表。

### 省额度的六个杠杆

| 手段 | 效果 |
|---|---|
| **按 Bot 分配模型** | 闲聊用便宜模型，只在写代码/研究时用贵模型。收益最大。 |
| `/compact` + 空闲压缩 | 长对话每轮都会重发全量历史。 |
| 关掉非必需的审核模型 | 每次审核都是一次额外的模型调用。 |
| 控制定时任务频率 | 每次触发都是一次完整对话。 |
| 群聊里少拉 Bot | 多个 Bot 各自回复 = 多次调用。`@` 谁才让谁说话。 |
| 用带上下文缓存的 provider | 部分服务商对缓存前缀打折很多。 |

### 回退

改之前先备份 profile 目录，回退时恢复并卸掉依赖：

```bash
cp -R ~/.dsh/profiles/<profile> ~/.dsh/profiles/<profile>.bak
# ...修改...
cp ~/.dsh/profiles/<profile>.bak/{cordis.patch.yml,package.json} ~/.dsh/profiles/<profile>/
cd ~/.dsh/profiles/<profile> && pnpm remove deepseekbot
```

### 绝对不要提交

- 真实 API Key（该放凭据存储，不该放 YAML）
- 公司内网网关地址或 IP
- `.credentials.yaml` 或任何凭据快照

## Files

| Path | Purpose |
|---|---|
| `examples/cordis.patch.example.yml` | Sanitized provider routes to paste into your profile patch |
| `docs/verification.md` | Exact commands used to prove the setup loads |
| `docs/troubleshooting.md` | Error-code reference from the pi-ai adapter |

## License

MIT — see [LICENSE](LICENSE).
