# 为什么我们在官方 v4.5.6 上打补丁，而不是升级官方包

本仓库是 [lbjlaq/Antigravity-Manager](https://github.com/lbjlaq/Antigravity-Manager) **v4.5.6** 的修补快照，不是官方发行渠道，也不是官方镜像的替代发布站。

我们只改了 Claude / Gemini / OpenAI 三条协议映射里 `thought_signature` 的处理。上游版权仍是官方的 **CC-BY-NC-SA-4.0**（本仓库的 `LICENSE` 已改回官方协议，覆盖 GitHub 建仓时的默认 MIT）。

## 本仓库包含什么 / 不包含什么

**包含**

- 官方 v4.5.6 源码快照（tag `v4.5.6`，commit `a2e3c45`）
- 当前生效的协议层补丁（工作树 = v4.5.6 + P0 + P1；P0 含 Flash 家族泛化并撤销哨兵；P1 补 OpenAI 兼容层签名透传）
- 补丁文件：`patches/`
- 可选的 Debian bookworm 后端构建文件：`docker/Dockerfile.backend.slim`

**不包含**

- 任何真实部署配置、域名、反代、机器地址
- 账号、OAuth token、API Key、管理密码
- 上游 GitHub Actions（`.github/workflows` 已去掉，避免在本仓库误触发官方的 Release / DockerHub 发布流程）
- 官方 `install.sh` / Cask 安装出来的**并不是**本补丁。那些脚本仍会去下载官方 GitHub Release。要用本补丁，必须从本仓库源码自行编译。
- 官方源码里硬编码的 Google OAuth 应用 `client_id` / `client_secret`（见下一节）

### 关于上游硬编码的 Google OAuth 凭据

官方 v4.5.6 的 `src-tauri/src/modules/oauth.rs` 把 Antigravity 桌面端的 Google OAuth 应用 id/secret 写死在常量里。那是**上游公开仓库里的凭据**，不是本快照维护者的密钥。GitHub 对本仓库启用了 push protection，原样推送会被拦截。

因此本快照改成从环境变量读取：

- `ANTIGRAVITY_OAUTH_CLIENT_ID`
- `ANTIGRAVITY_OAUTH_CLIENT_SECRET`
- 或官方已有的 `ANTIGRAVITY_OAUTH_CLIENTS`（`key|client_id|client_secret|label`）

无头网关（已有账号、只跑协议转换）不需要这两项。若要在本树编译的 GUI 里走 Google 登录，请从官方 v4.5.6 同文件自行填回，不要把值提交进 git。

---

## 1. 问题从哪来

Antigravity-Manager 把 Claude Code 的 Anthropic `/v1/messages` 转成 Google Gemini `v1internal`。Gemini 3.x（尤其是 Flash：3.5 / 3.6 / 3.7）对多轮 `functionCall` 有硬校验：

每个历史 `functionCall` 必须带 **这一轮 thinking 对应的** `thought_signature`。

上游会返回两类 HTTP 400，字面意思完全不同，修法也完全不同：

| 错误 | 含义 | 乱补会怎样 |
|---|---|---|
| `Function call is missing a thought_signature` | 缺签名 | 官方用哨兵 `skip_thought_signature_validator` 去「补上」 |
| `Invalid thought signature` | 签名在，但对不上这一轮 | 哨兵和「拿会话里最新那条签名盖上去」都会中招 |

Claude Code 会把完整历史（含 thinking 块和 tool_use）原样回放。网关如果在中途改掉 thinking、或给历史 tool_use 盖上别人的签名，Gemini 就报 Invalid，会话直接断。

这不是客户端 bug，是网关在协议转换时破坏了「thinking ↔ functionCall」的一一对应。

---

## 2. 第一轮：缺签名（官方 4.5.7 已经吸收）

官方 v4.5.6 把「无签名时仍保持 thinking」的白名单写死成：

- `gemini-3-flash`
- `gemini-3.1-flash`
- `gemini-pro-agent`

`gemini-3.7-flash` / `gemini-3.6-flash` / `gemini-3.5-flash` 都不在名单里。结果是：

1. 判定为「非 thinking 模型」，关掉 thinking
2. 历史 `functionCall` 不带 `thought_signature`
3. Gemini 返回 `400 Function call is missing a thought_signature`

第一轮补丁（`patches/thought-signature-gemini-3.7.patch`）做了两件事：

1. **泛化 Flash 家族**：`gemini` + `flash` 都视为需要签名的 thinking 模型
2. **缺签名时注入哨兵** `skip_thought_signature_validator`，让上游跳过「缺字段」校验

官方在 **v4.5.7**（2026-08-20，[PR #3314](https://github.com/lbjlaq/Antigravity-Manager/pull/3314)，修 [#3313](https://github.com/lbjlaq/Antigravity-Manager/issues/3313)）吸收了这一轮，changelog 原文就是「泛化 Flash 家族判断」+「哨兵签名兜底注入」。

第一轮能挡住「缺签名」。它**挡不住**「Invalid」。更糟的是：官方把哨兵当成正式方案保留了下来。

---

## 3. 为什么哨兵方案会变成 Invalid

`skip_thought_signature_validator` 只能骗过「字段缺失」检查。Gemini 3.x 的 Antigravity `agent` 路径一旦认为「这里应该有真实签名」，就会把哨兵和张冠李戴的真实签名一起判成 **Invalid thought signature**。

官方 4.5.6 / 4.5.7 / 4.5.8 里还有两条会主动制造「错签名」的路径。

### 3.1 Thinking-Sanitizer 在 family cache miss 时剥掉 thinking

`SignatureCache` 有 TTL（约 2 小时）和容量上限（family cache 约 200 条）。进程重启、TTL 过期、缓存挤掉之后，客户端回放过来的 thinking 签名在本地 cache 里查不到家族。

官方 `thinking_utils.rs` 的策略是：

```text
Dropping unverified signature (cache miss after restart)
```

直接把 thinking 块丢掉，「让上游重新生成」。

问题：Claude Code 下一轮仍会带回**已经配对过的**历史 `functionCall`。thinking 没了，functionCall 还在。后面的 mapper 只能去「找一个签名填上」，于是进入 3.2。

### 3.2 用「会话最新签名」或哨兵去填历史 functionCall

官方 `claude/request.rs` 的签名解析优先级包含：

1. 客户端自带
2. 当前上下文的 `last_thought_signature`（**会跨消息泄漏**）
3. `get_session_signature_at(session, msg_index)`（按轮次，这个是对的）
4. **`get_session_signature(session)`（会话里最新那一条）**
5. 全局 store（已废弃但仍作 fallback）
6. 都没有就注入 `skip_thought_signature_validator`

`gemini/wrapper.rs` 对 Flash 同样是：有 session latest 就盖 latest，没有就盖哨兵。

长会话里这会变成：

1. 早期 tool_use 对应较短 / 较旧的 thinking 签名
2. 缓存 miss 后 sanitizer 剥掉那些 thinking
3. mapper 把**最新一轮**的签名盖到**所有**历史 tool_use 上  
   或者盖上 `skip_thought_signature_validator`
4. Gemini：`400 Invalid thought signature`
5. 会话无法继续，只能新开 chat

第一轮的哨兵在这种情况下不是修复，是第二类 400 的触发器。

---

## 4. 第二轮 P0（官方 v4.5.8 仍未吸收）

第二轮补丁：`patches/thought-signature-invalid-p0.patch`。

原则：**宁可缺签名（用户可以新开会话），不要伪造签名（会毒化当前轮次）。**

| 文件 | 官方行为（至 v4.5.8） | 本仓库 |
|---|---|---|
| `src-tauri/src/proxy/mappers/claude/thinking_utils.rs` | family cache miss → **丢掉** thinking | cache miss 时 **保留** 客户端回放的 thinking |
| `src-tauri/src/proxy/mappers/claude/request.rs` | 跨消息沿用 `last_thought_signature`；fallback 到 latest session / 全局 store / skip 哨兵 | **每条消息重置** `last_thought_signature`；只接受「本条 tool_use 自带 / 同一条 assistant 消息里的 thinking / tool_id cache / `get_session_signature_at`」；无签名就省略字段 |
| `src-tauri/src/proxy/mappers/gemini/wrapper.rs` | Flash 无签名时注入 latest 或 skip 哨兵 | 无签名则 `Leaving functionCall unsigned`，不盖 latest、不盖哨兵 |
| `src-tauri/src/proxy/mappers/openai/request.rs` | 占位 thinking / 无签名 tool_use 注入 skip 哨兵；P0 后改为整段省略 | P1：按 tool_call 回放真实签名，见下一节 |

签名绑定优先级（Claude 映射，P0 之后）：

```text
tool_use.signature
  → 同一条 assistant 消息里的 thinking 签名
  → SignatureCache.get_tool_signature(tool_id)
  → SignatureCache.get_session_signature_at(session, msg_index)
  → 省略 thought_signature 字段
```

明确不做的事：

- 不用 `get_session_signature()`（latest）
- 不用全局 thought signature store
- 不注入 `skip_thought_signature_validator`
- 不把 thinking 签名盖到 `functionResponse` 上

P0 已经带上 Flash 家族泛化，因此不必再叠第一轮 patch（两份都是相对 v4.5.6 的完整 diff，顺序 apply 会冲突）。第一轮文件只作历史对照。

---

## 4.1 第三轮 P1：OpenAI 兼容层签名透传

补丁：`patches/thought-signature-openai-p1.patch`（相对本仓库 P0 快照，不是相对官方裸 v4.5.6）。

P0 把 OpenAI 路径也改成「无签名就省略、不用 latest」。这对 Claude Code 是对的，因为 Anthropic 消息里会带回 thinking。走 **OpenAI `/v1/chat/completions`** 的客户端（例如 DSH）没有标准字段装 `thoughtSignature`，历史 `functionCall` 在网关侧就会变成无签名。

Gemini 3.x Flash 这时不一定立刻 400。官方说明是 missing thought_signature **may lead to degraded model performance**：后续工具调用漏必填参数（`bash` 只剩 `description`、`write` 空参）。会话中断几小时后更明显——原先 session/tool 缓存 TTL 只有 2 小时，过期后整段历史全部 `Omitting thought_signature`。

P1 做了三件事：

1. **响应侧透传**：流式 / 非流式把签名写到 `tool_call.thought_signature` 以及 `extra_content.google.thought_signature`，并按网关生成的 `call_*` id 写入 tool cache。
2. **请求侧按条回放**：客户端带回的字段 → `get_tool_signature(tool_id)` → `get_session_signature_at(session, msg_index)` → 同一条 assistant 消息里已绑定的签名。仍然 **不用** `get_session_signature()` latest，也 **不** 注入 skip 哨兵。
3. **回放缓存 TTL**：tool / session 签名从 2 小时改为 **7 天**（family cache 仍是 2 小时）。

当前工作树 = **官方 v4.5.6 + P0 + P1**。

---

## 5. 对照官方版本：为什么先不升级

核查日期：2026-08-22。官方最新 tag 当时是 **v4.5.8**。

| 官方版本 | 发布 | 和 thought_signature 的关系 | 能不能当本补丁的替代 |
|---|---|---|---|
| **v4.5.6** | 2026-08-15 | 本仓库基线。Flash 白名单写死 3 / 3.1 | 基线 |
| **v4.5.7** | 2026-08-20 | [PR #3314](https://github.com/lbjlaq/Antigravity-Manager/pull/3314) 吸收第一轮（Flash 泛化 **+ 哨兵兜底**）。另有账号 5H/周配额视图、智能预热，与签名无关 | **不能**。哨兵正是 Invalid 的来源之一 |
| **v4.5.8** | 2026-08-22 | [PR #3316](https://github.com/lbjlaq/Antigravity-Manager/pull/3316) 只做 Claude Agent SDK / CC GUI 身份归一（修 503 `RESOURCE_EXHAUSTED`），与签名无关 | **不能** |

v4.5.8 源码抽查（官方 tag，不是 changelog）：

- `thinking_utils.rs` 仍是 `Dropping unverified signature (cache miss after restart)`
- `claude/request.rs` 仍有 `Recovered latest signature from SESSION cache`，仍注入 `skip_thought_signature_validator`
- `gemini/wrapper.rs` 对 Flash 仍注入 latest 或 skip 哨兵

因此：把运行中的镜像换成官方 `latest` / `v4.5.8`，第一轮还在，**第二轮会退回去**，`Invalid thought signature` 会再现。

v4.5.8 的 503 修复只影响 Claude Agent SDK / CC GUI 那种独立身份声明（`"You are a Claude agent, built on Anthropic's Claude Agent SDK."`）。Claude Code CLI 本身已经是官方 CLI 身份，吃不到这条。配额视图、智能预热也不值得为此回退签名修复。

正确的升级方式（以后若要做）：把 P0 **rebase 到官方新 tag**，再编译，而不是直接拉官方镜像。

---

## 6. 建议镜像 tag

源码版本字符串仍是上游的 `4.5.6`（健康检查会报 `{"status":"ok","version":"4.5.6"}`）。为避免和官方 Docker Hub 的 `4.5.6` 混淆，本地镜像请打自己的 tag，例如：

```text
antigravity-manager:local-4.5.6-sigfix
```

不要把本仓库描述成「官方 4.5.6」。它是 **4.5.6 + thought_signature P0 + OpenAI P1**。

---

## 7. 怎么从本仓库构建

本仓库**没有**提供我们自己的运行镜像或部署说明。下面只给通用编译提示。

协议补丁全在 Rust 后端。无头网关二进制是 `antigravity_tools`。

若运行时是 Debian bookworm（glibc 2.36）容器，不要在更新的 glibc 主机上直接 `cargo build` 再拷进去，否则会报 `GLIBC_2.39 not found`。用 bookworm 工具链编：

```bash
# 示例：docker/Dockerfile.backend.slim
# 基于 rust:1-slim-bookworm，产物可在 debian:bookworm-slim 里跑
docker build -f docker/Dockerfile.backend.slim -t antigravity-manager:local-4.5.6-sigfix .
```

`Dockerfile.backend.slim` 只是为了在 bookworm 上编后端、并复用已有前端 dist 镜像；它不包含任何私有仓库地址。前端占位、`ABV_DIST_PATH` 等行为与官方 headless 一致。

针对性回归（补丁相关）：

```bash
cd src-tauri
cargo test --lib skip_thought_signature_validator
cargo test --lib openai_replays_client
```

官方原有测试里有「应当注入哨兵」的用例，P0 已改成「不得注入哨兵」。P1 另有客户端回放 / tool-id 缓存测试。

---

## 8. 以后何时可以丢掉这份补丁

同时满足再考虑跟官方：

1. 官方 `thinking_utils.rs` 在 family cache miss 时保留客户端回放的 thinking，而不再 `Dropping unverified signature`
2. 官方 Claude / Gemini / OpenAI mapper **不再**用 latest session 签名去填历史 `functionCall`
3. 官方 **不再**注入 `skip_thought_signature_validator`（至少在 Antigravity `agent` / Gemini 3.x Flash 路径上）
4. changelog 或 issue 明确覆盖 `Invalid thought signature`，而不只是 `missing a thought_signature`

在此之前，本仓库停留在 **v4.5.6 + P0**。

---

## 9. 文件清单

| 路径 | 作用 |
|---|---|
| `patches/thought-signature-gemini-3.7.patch` | 第一轮历史补丁（缺签名 / Flash 白名单）。不要再 apply 到当前树 |
| `patches/thought-signature-invalid-p0.patch` | 第二轮 P0，相对官方 v4.5.6 |
| `patches/thought-signature-openai-p1.patch` | 第三轮 P1，相对本仓库 P0 快照：OpenAI 签名透传 + 7 天回放缓存 |
| `patches/README.md` | 补丁使用说明 |
| `src-tauri/src/proxy/mappers/claude/request.rs` | Claude → Gemini 签名绑定 |
| `src-tauri/src/proxy/mappers/claude/thinking_utils.rs` | sanitizer：cache miss 保留 thinking |
| `src-tauri/src/proxy/mappers/gemini/wrapper.rs` | 原生 Gemini 包装：无签名则保持 unsigned |
| `src-tauri/src/proxy/mappers/openai/request.rs` | OpenAI 请求：按 tool_call 回放签名 |
| `src-tauri/src/proxy/mappers/openai/streaming.rs` / `response.rs` / `collector.rs` / `models.rs` | OpenAI 响应把签名交给客户端 |
| `src-tauri/src/proxy/signature_cache.rs` | tool/session 回放 TTL 7 天 |
| `docker/Dockerfile.backend.slim` | 可选：bookworm 后端构建 |
| `src-tauri/src/modules/oauth.rs` | 去掉上游硬编码 OAuth 常量，改环境变量（仅为了能推送到 GitHub） |
