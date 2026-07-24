# 06. 模型供应商与协议适配

## 1. 三个不能混用的概念

`docs/architecture/integrations.md` 明确区分：

- **metadata**：route 是什么、默认模型、标签、能力、认证提示；
- **routing**：当前 settings/env/profile/model 映射到哪个 route；
- **transport**：实际 HTTP/SDK 请求和 stream 解析。

新增 provider 时，显示信息应进入 descriptor；真实协议差异进入 transport；兼容旧环境变量的逻辑留在 compatibility bridge。

## 2. Descriptor-first 集成模型

`src/integrations/` 的实体：

- vendor：模型厂商；
- gateway：提供 API 路由的服务；
- brand：UI/模型系列归类；
- model：共享能力与上下文元数据；
- anthropic proxy：兼容 Anthropic messages 的代理；
- route/catalog：某入口可用的模型集合。

正常 descriptor 使用 `define*` + default export，generator 生成 artifacts，`integrations/index.ts` 懒注册到 registry。

`transportConfig.kind` 决定 runtime family；gateway `category` 只做展示，不能用于协议路由。

## 3. 配置入口与优先级

模型/provider 来源可能包括：

- CLI `--provider`、`--model`；
- active provider profile；
- settings 默认 model/provider；
- provider 环境变量；
- skill/agent/tool 指定 model；
- smart routing 的 per-turn 选择；
- agent routing 的跨 provider override；
- fallback model/provider chain。

需要区分：

- 主 session model；
- 某次 turn pinned model；
- 子代理 resolved model；
- provider profile 的 primary model；
- transport 最终使用的 API model name。

`parseUserSpecifiedModel()` 和 provider resolver 会处理 alias；descriptor catalog 映射 UI id 与 API name。

## 4. Provider Profile

profile 把 route 所需环境集中保存，例如 base URL、API key、model、API format、额外 headers。激活 profile 时：

1. 清理上一 provider 的互斥环境变量；
2. 应用新 profile 的 env-facing contract；
3. 设置 active profile id；
4. 模型 resolver 使用 profile primary model；
5. UI/StartupScreen 显示当前路由。

profile 是当前 public env contract 的兼容桥，因此运行时仍能看到显式 env 分支。

## 5. 传输家族

### 5.1 Anthropic native

适用于第一方 Anthropic 及 Anthropic-family 云路径。保留：

- Messages API content blocks；
- thinking/redacted thinking signature；
- prompt cache controls；
- native web search（仅支持路径）；
- Anthropic usage/stop reason。

Bedrock、Vertex、Foundry 虽属 Anthropic-family，但认证/client 构造不同，不应走 generic OpenAI shim。

### 5.2 OpenAI-compatible

`services/api/openaiShim.ts` 把内部 Anthropic-like 请求语义映射为：

- Chat Completions；
- Responses API；
- Responses compatibility 变体。

职责包括：

- role/content/tool schema 转换；
- tool choice 和参数 schema 清理；
- streaming delta 还原成内部 blocks；
- reasoning content 兼容；
- usage、finish reason、error category 映射；
- 某些模型输出 XML/text tool call 的容错解析；
- provider-specific header/auth/request shaping。

不能假设所有兼容接口完全一致。DeepSeek、Kimi/Moonshot、Mistral、Azure、Bankr、Ollama 等仍有实质例外。

### 5.3 Codex/OpenAI Responses

`codexShim.ts` 处理 Codex/OpenAI Responses 特有语义、OAuth 和工具协议；GitHub 路由可能根据模型在 native Anthropic 与 Codex/OpenAI 路径间切换。

### 5.4 Gemini

Gemini 需要单独处理 credential、content/part、function calling、thought signature 和 Vertex Gemini client。它不是只改 base URL 的 OpenAI-compatible route。

## 6. API Client 构造

`services/api/client.ts` / `providerConfig.ts` 负责：

- 根据 route/provider 选择 client；
- 读取 API key/OAuth/cloud credential；
- base URL、proxy、TLS/cert；
- headers 和 beta；
- timeout、fetch 实现和连接复用；
- session ingress/remote 认证；
- optional runtime modules 缺失时给出明确错误。

认证来源可能是 env、keychain、OAuth refresh、helper command、AWS/GCP default chain 或 SDK 注入。

## 7. API 请求正规化

在发送前：

1. internal messages 去除 UI-only 字段；
2. 校验 tool use/result pairing；
3. 根据 provider 处理 thinking blocks；
4. 应用 system/user context；
5. 构造当前工具 schema；
6. 决定 max tokens、thinking/effort、temperature 等；
7. 应用 prompt cache、beta、metadata；
8. 传给对应 transport。

不同 provider 不支持的字段必须删除，不能原样透传。

## 8. 流式响应统一

上层 `query.ts` 不应该理解每个 provider 的原始 SSE。transport 将其转为内部事件：

- message start；
- content block start/delta/stop；
- text/thinking/tool JSON delta；
- usage；
- completed/error。

工具 JSON 参数可能跨多个 delta；必须累积完成后再校验。错误出现在 stream 中时也应转为 synthetic assistant API error，而不是随意 throw；throw 仅作为未预期 runtime failure 的保底。

## 9. Smart Routing

`services/api/smartRouting/` 可按最新 user text、非文本内容和配置决定该 turn 使用 simple 或 strong model。关键语义：

- 一次 user turn 的决定被 pinned，后续 tool continuation 不重复分类；
- simple route 遇到可重试模型错误，可一次升级 strong；
- abort 和 4xx client/auth/bad request 不升级；
- fallback 时清理旧 tool IDs 和不兼容 thinking；
- 记录 decision 与 escalation telemetry。

## 10. Agent Routing

`services/api/agentRouting.ts` 支持按 `agentName` 或 `subagent_type` 配置：

- model-only：复用父 provider，只换模型；
- full provider override：base URL、key、model 和 provider env 清理。

优先级会考虑 tool 显式 model、agent definition model、settings route、父模型和 permission mode。组织 model allowlist 在最终 resolved model 上检查。

out-of-process teammate 不能共享内存 client，因此把 override 转成 CLI 参数/子进程 env；in-process subagent 则在 `runAgent()` 内局部解析。

## 11. Fallback 的三种层次

### 11.1 API retry

同 provider/client 的瞬时故障：连接、408、409、允许的 429、5xx。指数退避并尊重 Retry-After。

### 11.2 Model fallback

连续 529 达阈值且配置 `fallbackModel`：`withRetry` 抛 `FallbackTriggeredError`，query 清理第一次 attempt 后换模型重发。

### 11.3 Provider fallback chain

最终 assistant error 为 rate_limit 时，query 按 settings 的 profile id 顺序切换 active provider；更新 session model，再重试一次。compact/session-memory fork 不允许中途改外层 provider。

这三层必须分开，否则会重复 retry 或把错误模型 id 发到新 endpoint。

## 12. 错误映射

`services/api/errors.ts` 将底层异常转为可操作信息：

- invalid/revoked key；
- credit/quota；
- overloaded/rate limit；
- prompt/media too large；
- vision unsupported；
- provider max token cap；
- OpenAI compatibility category；
- Bedrock/Vertex auth；
- OpenCode Go 配额等。

还保留原始分类用于 telemetry/SDK。用户文案不能替代 machine-readable `error` 字段。

## 13. Usage 与成本

assistant usage 累计到 bootstrap state，按 model 分组：

- input/output；
- cache creation/read；
- web search requests；
- cost 与 API duration。

未知模型价格设置 unknown flag，避免给出虚假精确成本。某些 provider 有独立 usage endpoint（如 MiniMax/Codex），不能完全从响应 token 推导余额。

## 14. Provider 扩展检查表

1. 确认是 vendor、gateway 还是 proxy。
2. 添加 descriptor/catalog，不复制列表。
3. 选择正确 `transportConfig.kind`。
4. 只有真实协议差异才改 transport。
5. 定义 auth/env/profile UI 元数据。
6. 验证 model alias、context window、vision/tool/reasoning 能力。
7. 测试 request shape、stream、tool call、error、usage。
8. 运行 `integrations:generate` 和 `integrations:check`。
9. 验证不会破坏第三方兼容 provider。

下一章：[07 工具、权限、沙箱与文件安全](07-tools-permissions-security.md)。
