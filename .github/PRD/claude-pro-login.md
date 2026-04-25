# PRD：InkOS 支持 Claude Pro / Max 订阅登录

| 字段 | 内容 |
| --- | --- |
| 状态 | Draft |
| 作者 | InkOS Maintainers |
| 创建日期 | 2026-04-25 |
| 目标分支 | `claude/add-claude-pro-login-EVpWN` |
| 文档路径 | `.github/PRD/claude-pro-login.md` |
| 关联 | `packages/core/src/llm/provider.ts`、`packages/core/src/llm/secrets.ts`、`packages/cli/src/commands/`、`packages/studio/src/api/server.ts` |

---

## 1. 背景

InkOS 当前通过 OpenAI 兼容协议或 Anthropic Messages API 调用 LLM，认证方式**仅支持 API Key**：

- 环境变量：`INKOS_LLM_API_KEY` / `ANTHROPIC_API_KEY`
- 持久化：`./.inkos/secrets.json`
- 调用现场：`packages/core/src/llm/provider.ts:532-540` 同时塞 `x-api-key` 与 `Authorization: Bearer <api_key>`（冗余，但目前对 API Key 模式无害）

订阅了 Claude **Pro** 或 **Max** 的用户拥有可观的对话额度，但这部分额度无法直接用于 API 接入——他们需要再去 Anthropic Console 充值，再申请独立的 API Key，体验割裂、成本翻倍。社区（craft-agents-oss、opencode-claude-auth、weidwonder/claude_agent_sdk_oauth_demo 等）已经验证：用 Claude Code 内部使用的 OAuth 流程，可以让 Pro/Max 订阅者直接复用订阅额度调用 `api.anthropic.com`。

## 2. 目标 / 非目标

### 2.1 目标
1. 让持有 Claude Pro / Max 订阅的用户在 InkOS 中**用浏览器登录一次**，就能跑完所有需要 Anthropic 模型的 agent 流水线。
2. 同时支持 **CLI（含 TUI 引导）** 与 **Studio 桌面/Web UI** 入口。
3. 与现有 API Key 模式**并存**：用户可在两种模式间切换，旧用户零迁移成本。
4. 凭证安全：`access_token` / `refresh_token` 写入 `.inkos/secrets.json`，文件权限收紧（POSIX `0600`）。
5. 自动续期：`access_token` 临近过期前自动刷新，对上层透明。

### 2.2 非目标
- **不**替代 API Key 模式（团队、CI、其他模型仍需 API Key）。
- **不**为 OpenAI / 兼容服务实现 OAuth（仅 Anthropic）。
- **不**在仓库代码中硬编码官方 client_id（见 §9 合规）。
- **不**实现多账户切换（v1 单账户，多账户列入 v2）。

## 3. 用户故事

| ID | 角色 | 故事 |
| --- | --- | --- |
| US-1 | Pro 订阅者 | 跑 `inkos login claude` → 浏览器自动打开 Anthropic 授权页 → 授权后复制返回码 → 终端粘贴 → 看到 "✓ Logged in as <email>" → 后续 `inkos write` 不再要求填 API Key |
| US-2 | Max 订阅者 | 在 Studio 设置页点击「使用 Claude Pro/Max 登录」→ 浏览器跳转 → 复制粘贴 → 设置页显示当前账号与剩余额度提示 |
| US-3 | 现有 API Key 用户 | 不做任何动作，旧的 `INKOS_LLM_API_KEY` 工作流不受影响 |
| US-4 | 长时间运行的用户 | 一次大型 `compose` 跑了 4 小时，期间 token 自动续期，没有任何中断 |
| US-5 | 多设备开发者 | 把 `~/.claude/.credentials.json` 拷过去后无需重登（如启用 §6.3 兼容读取） |

## 4. 现状代码触点

> **必读**：以下文件路径与行号是新功能必须修改/读取的位置，PR 评审时按图索骥。

| 文件 | 当前用途 | 本次需要做什么 |
| --- | --- | --- |
| `packages/core/src/llm/provider.ts:532-608` | Anthropic Messages API 调用 | header 改造：根据认证模式分支，OAuth 模式去掉 `x-api-key`，加 `anthropic-beta: oauth-2025-04-20`，附带 User-Agent |
| `packages/core/src/llm/secrets.ts:1-65` | 读写 `secrets.json`（仅 `apiKey`） | 扩展 `SecretsFile`，新增 `oauth` 字段（见 §6.1）；写文件时 `fs.chmod(0o600)` |
| `packages/core/src/llm/providers/endpoints/anthropic.ts` | Anthropic 端点配置 | 暴露认证方式枚举，供上游分支选择 |
| `packages/core/src/utils/effective-llm-config.ts` | 配置合并 | 加入 oauth/api-key 优先级解析（见 §7） |
| `packages/cli/src/tui/setup.ts` | TUI 首次设置 | 增加「使用 Claude Pro/Max 登录」分支 |
| `packages/cli/src/commands/config.ts` | `inkos config` 命令 | 增加 `--auth-mode oauth\|api-key` 切换 |
| `packages/cli/src/commands/doctor.ts:250-258` | 健康检查 | 检测 oauth token 是否过期 / 是否 refresh 可用 |
| `packages/cli/src/commands/` (新增 `login.ts`、`logout.ts`) | — | 实现 OAuth 流程入口 |
| `packages/studio/src/api/server.ts:295-298, 681` | 后端 LLM 配置加载 | 加 `/api/auth/claude/start` 与 `/api/auth/claude/finish` 路由 |
| `packages/studio/src/...`（前端设置页） | 当前只有 API Key 输入 | 增加「Claude Pro/Max 登录」按钮 |

## 5. OAuth 技术规范

### 5.1 端点（基于社区共识，可被官方文档覆盖）

| 名称 | URL |
| --- | --- |
| 授权 | `https://claude.ai/oauth/authorize` |
| Token 交换 / 刷新 | `https://console.anthropic.com/v1/oauth/token` |
| 复制粘贴回调（无本地 server） | `https://console.anthropic.com/oauth/code/callback` |
| 客户端 ID | `9d1c250a-e61b-44d9-88ed-5944d1962f5e`（Claude Code 公开默认值；详见 §9） |
| 推荐 scope | `org:create_api_key user:profile user:inference` |
| 最小 scope | `user:inference` |

### 5.2 PKCE（S256）

```ts
function base64url(buf: Buffer) {
  return buf.toString("base64").replace(/\+/g, "-").replace(/\//g, "_").replace(/=+$/, "");
}
const verifier  = base64url(crypto.randomBytes(32));
const challenge = base64url(crypto.createHash("sha256").update(verifier).digest());
```

### 5.3 授权 URL

```
https://claude.ai/oauth/authorize
  ?response_type=code
  &client_id=9d1c250a-e61b-44d9-88ed-5944d1962f5e
  &redirect_uri=https%3A%2F%2Fconsole.anthropic.com%2Foauth%2Fcode%2Fcallback
  &scope=org%3Acreate_api_key%20user%3Aprofile%20user%3Ainference
  &code_challenge=<challenge>
  &code_challenge_method=S256
  &state=<random>
```

### 5.4 用户回调码格式

页面会显示形如 `<code>#<state>` 的字符串。InkOS 收到后必须按 `#` 分割：

```ts
const [code, returnedState] = pasted.split("#", 2);
if (returnedState !== state) throw new Error("OAuth state mismatch");
```

> 若仅返回 `<code>`，按 ben-vargas gist 的 fallback：把 PKCE verifier 当 state 再传一次。

### 5.5 Token 交换

`POST https://console.anthropic.com/v1/oauth/token`，`Content-Type: application/json`：

```json
{
  "grant_type": "authorization_code",
  "code": "<code>",
  "client_id": "9d1c250a-e61b-44d9-88ed-5944d1962f5e",
  "redirect_uri": "https://console.anthropic.com/oauth/code/callback",
  "code_verifier": "<verifier>",
  "state": "<state>"
}
```

返回示例：

```json
{
  "access_token": "...",
  "refresh_token": "...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "scope": "user:inference user:profile"
}
```

### 5.6 刷新

距离 `expiresAt` 小于 **5 分钟**时触发；并发请求合并到同一 promise（避免雷鸣群）：

```ts
POST https://console.anthropic.com/v1/oauth/token
{
  "grant_type": "refresh_token",
  "refresh_token": "<refresh_token>",
  "client_id": "9d1c250a-e61b-44d9-88ed-5944d1962f5e"
}
```

刷新失败（401 / `invalid_grant`）→ 清除凭证，提示用户重新 `inkos login claude`。

## 6. 调用 Anthropic Messages API 的 Header（关键风险）

### 6.1 OAuth 模式下的目标 header

```
POST https://api.anthropic.com/v1/messages
Authorization: Bearer <access_token>
anthropic-version: 2023-06-01
anthropic-beta: oauth-2025-04-20
User-Agent: inkos/<version> (claude-cli compatible)
Content-Type: application/json
```

**必须删除** `x-api-key`（OAuth 模式下保留它会导致 Anthropic 报 400 / 401，与生态多份 issue 一致）。

### 6.2 已知争议

| 来源 | 观点 |
| --- | --- |
| Claude Code 主线 / craft-agents-oss / opencode-claude-auth (2026.04 仍在用) | `Authorization: Bearer` + `anthropic-beta: oauth-2025-04-20` |
| pi-mono#2751 报告 | 某段时间 Anthropic 改成只接受 `x-api-key`（即把 access_token 放 `x-api-key`），Bearer 被 401 |
| anthropic/claude-code#13770 | 部分版本对 `oauth-2025-04-20` 报 "Unexpected value" |

**实现要求**：
1. 默认走 §6.1 方案；
2. 暴露内部环境变量 `INKOS_OAUTH_HEADER_MODE=bearer|x-api-key` 作为应急开关；
3. 失败时清晰地把 Anthropic 返回的 error body 带回用户面前。

### 6.3 User-Agent

不要伪造成 `claude-cli/x.y.z`（社区已观察到 429 风控收紧）。建议 `inkos/<version> (claude-cli compatible)`，既声明来源又避免被认成 abuse。

## 7. 凭证存储模型

### 7.1 类型变更（`packages/core/src/llm/secrets.ts`）

```ts
export interface OAuthCredential {
  provider: "anthropic";
  accessToken: string;
  refreshToken: string;
  expiresAt: number;          // 毫秒级 unix 时间戳
  scopes: string[];
  obtainedAt: number;
  accountEmail?: string;      // 仅展示用，可空
}

export interface ServiceCredential {
  apiKey?: string;
  oauth?: OAuthCredential;
}

export interface SecretsFile {
  services: Record<string, ServiceCredential>;
}
```

### 7.2 文件权限

- 写入后立即 `await chmod(path, 0o600)`（POSIX）。
- Windows 不动 ACL（v1 不处理）。
- `.inkos/` 已在 `.gitignore` 中，无需新增。

### 7.3 兼容读取 Claude Code 凭证（可选，强烈推荐）

如果 `~/.claude/.credentials.json` 存在且 InkOS 自身没有凭证，**只读**地复用其中的 `claudeAiOauth` 字段，避免用户重复登录。复用时不写回该文件，刷新得到的新 token 写到 InkOS 自己的 `.inkos/secrets.json`。

## 8. CLI / TUI / Studio 用户流程

### 8.1 CLI

```
$ inkos login claude
→ 浏览器自动打开授权 URL（用 `open` / `xdg-open` / `start`，失败时回落到打印链接）
→ 终端提示：「请粘贴页面显示的授权码（CODE#STATE）：」
→ 用户粘贴 → 内部交换 token → 写入 .inkos/secrets.json
→ 输出：✓ Logged in (scopes: user:inference, user:profile)

$ inkos logout claude
→ 删除 secrets.services.anthropic.oauth → 输出确认
```

### 8.2 TUI 首次设置

`packages/cli/src/tui/setup.ts` 在「选择 LLM Provider」之后加分支：

```
Provider: Anthropic
  ❯ 使用 Claude Pro/Max 订阅登录   (需浏览器)
    使用 API Key
```

### 8.3 Studio

后端：

| 路由 | 行为 |
| --- | --- |
| `POST /api/auth/claude/start` | 生成 verifier/challenge/state，缓存到内存（带 10 分钟 TTL），返回授权 URL |
| `POST /api/auth/claude/finish` | 接收前端粘贴的 `code#state` → 交换 token → 写 secrets → 返回账号信息 |
| `POST /api/auth/claude/logout` | 清除 oauth 字段 |
| `GET  /api/auth/claude/status` | 返回是否登录、scopes、过期时间 |

前端：设置页加「使用 Claude Pro/Max 登录」按钮 + 状态展示。

## 9. 合规与风险

> **InkOS 必须明确这是社区互操作方案，不是 Anthropic 官方支持的接入方式。**

1. **2026 年 3 月事件**：Anthropic 向 OpenCode 发出法务函，要求其停止分发预置 OAuth 客户端。InkOS 的对策：
   - **不在仓库源码里硬编码** `client_id`。建议：首次运行 `inkos login claude` 时检测本地 `~/.claude/.credentials.json`，若存在直接复用；否则从远端配置或用户输入获取 client_id（默认值仅写在 PRD/文档里，作为知识，不入二进制）。
   - 仓库 README 加显式声明：「本功能基于公开 OAuth 协议互操作；Anthropic 未承诺支持订阅额度用于第三方工具，使用风险自担。」
   - 默认**关闭**该功能，需要用户主动 opt-in（环境变量 `INKOS_ENABLE_CLAUDE_OAUTH=1` 或交互确认）。
2. **速率与额度**：Pro/Max 通过 OAuth 调用受订阅级速率限制，超限时返回 429。需要把 Anthropic 错误体（含 `retry-after`）原样透传给上层 agent，并在 TUI/Studio 给出友好文案。
3. **凭证泄漏**：`access_token` 一旦泄漏可被用于消耗用户订阅额度。强制 `0600` + 写入前后做 `JSON.parse` 校验防止文件被破坏。
4. **不保证长期可用**：Anthropic 可在任意版本调整 header 校验或停止 OAuth 模式。`provider.ts` 的错误处理需把这种情况识别为「需要用户重新登录或切回 API Key」。

## 10. 配置优先级

`effective-llm-config.ts` 决议顺序（高 → 低）：

1. CLI flag `--auth-mode oauth|api-key`（强制）
2. `secrets.services.anthropic.oauth`（若未过期或可刷新）
3. `secrets.services.anthropic.apiKey`
4. 环境变量 `ANTHROPIC_API_KEY` / `INKOS_LLM_API_KEY`

匹配到任一项即停止。

## 11. 测试矩阵

| 场景 | 预期 |
| --- | --- |
| 仅 API Key | 与现状一致，全部测试通过 |
| 仅 OAuth，token 未过期 | Anthropic 调用成功 |
| OAuth + API Key 共存 | 默认走 OAuth |
| `--auth-mode api-key` 强制覆盖 | 改走 API Key |
| access_token 过期但 refresh_token 有效 | 自动刷新，请求成功 |
| refresh_token 失效 | 清凭证 + 提示重登 |
| Anthropic 返回 429 | 透传 `retry-after`，上层退避 |
| 网络中断 / DNS 失败 | 错误消息明确说明阶段（authorize / token / message） |
| 文件权限被改成 0644 | 启动时 warning 并自动收紧到 0600 |

E2E：`packages/cli/src/__tests__/` 增 `login.test.ts`，使用 `nock` 拦截两个端点。

## 12. 实现路线图

| 里程碑 | 内容 |
| --- | --- |
| **M1** | `secrets.ts` 扩展 + CLI `login` / `logout` + token 交换（无 UI 集成） |
| **M2** | `provider.ts` header 路由 + 自动刷新 + `doctor.ts` 检查 |
| **M3** | TUI 集成 |
| **M4** | Studio 后端路由 + 前端按钮 + 状态展示 |
| **M5** | 兼容读取 `~/.claude/.credentials.json` + 文档与 README 声明 |

## 13. 未决问题（实现期决策）

1. User-Agent 是否拼上 InkOS 标识？倾向"是"（透明 + 防风控）。
2. 是否提供 loopback redirect (`http://127.0.0.1:<port>/callback`) 替代复制粘贴？UX 更好但增加 firewall 风险。v1 暂用复制粘贴，v2 评估。
3. client_id 来源：硬编码在 PRD 但运行期从配置读，还是允许用户自带 client_id？倾向后者（更稳健）。
4. Studio 是否需要把登录二维码生成给手机端？v2 再说。

## 14. 外部参考

- [lukilabs/craft-agents-oss](https://github.com/lukilabs/craft-agents-oss) — 桌面端 Claude Pro/Max OAuth 完整实现（`packages/shared/src/auth/`）
- [weidwonder/claude_agent_sdk_oauth_demo](https://github.com/weidwonder/claude_agent_sdk_oauth_demo) — Claude Agent SDK 用 Pro 账户最小 demo
- [griffinmartin/opencode-claude-auth](https://github.com/griffinmartin/opencode-claude-auth) — header / token 刷新 / Keychain 兼容实现细节
- [ex-machina-co/opencode-anthropic-auth](https://github.com/ex-machina-co/opencode-anthropic-auth) — 另一个 OpenCode 侧 OAuth 插件
- [ben-vargas / claude-code-sdk_oauth.md](https://gist.github.com/ben-vargas/c7c7cbfebbb47278f45feca9cef309d1) — 端点 / PKCE / 存储格式权威总结
- 已知 Issue：
  - `anthropics/claude-code#13770` — `oauth-2025-04-20` 被部分版本拒绝
  - `badlogic/pi-mono#2751` — Anthropic 拒 Bearer，要求 `x-api-key`
  - `BerriAI/litellm#22398` — anthropic-beta header 被 OAuth handler 误覆盖
  - `RooCodeInc/Roo-Code#4799` — 同类需求讨论

## 15. 验收标准

- 一名工程师按本 PRD 可独立实现该功能，无需再读 craft-agents-oss 源码即可完成 M1。
- README / `inkos --help` / Studio 设置页都明确声明「非官方」与「自担风险」。
- 旧 API Key 用户升级后零迁移成本（自动测试覆盖）。
- `inkos doctor` 能识别 oauth 已过期且 refresh 失败的状态。
