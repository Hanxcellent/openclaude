# 12. 错误、重试、恢复与特殊场景

OpenClaude 不用一个全局 `catch` 处理所有故障。错误在接近事实来源的位置分类，再由 API、query、tool、UI 或 shutdown 层执行不同恢复策略。

## 12.1 错误处理分层

```text
provider/transport 原始异常
  -> provider 兼容层分类
  -> withRetry 判断可重试性
  -> 转换为 Assistant API error message
  -> query 状态机决定压缩、降额、fallback 或结束
  -> UI/SDK 按结构化 message 展示
```

工具、MCP、hook 和进程异常使用各自相邻的分类器。所有分类器遵循同一原则。可预期业务失败形成结构化结果。不可恢复的程序错误抛到上层。

## 12.2 Provider 错误规范化

第一方 Anthropic SDK、Bedrock、Vertex、OpenAI-compatible 服务的异常形态不同。`src/services/api/errors.ts` 将其转成统一的 assistant error message，至少区分：

- authentication、permission、billing/quota。
- rate limit、overloaded、timeout、network。
- context overflow、max output tokens。
- model/endpoint not found。
- image/PDF 尺寸或页数错误。
- tool calling 不兼容、`tool_stream` 不支持。
- malformed provider response。

OpenAI-compatible 层把分类写入内部 marker，例如 `openai_category=context_overflow`。用户文案会移除 marker。retry/query 层可以稳定读取该分类。该机制消除了对自然语言错误文案的二次解析。

可自动重试的 OpenAI-compatible 类别限于连接失败、DNS/localhost、请求超时、网络、rate limit 和 provider unavailable。认证失败、模型不存在、工具协议不兼容、配额耗尽等需要用户改变配置，不能靠重复请求解决。

## 12.3 API Retry 策略

`withRetry()` 是 API 调用的有限重试器。默认最多 10 次 retry，基础延迟 500 ms，指数退避并加入最多 25% jitter。普通路径的退避基数封顶约 32 秒，合法 `Retry-After` 优先。

### 会重试的典型情况

- 429、部分 5xx、连接超时和网络异常。
- AWS/GCP 临时凭据错误。先清凭据缓存。
- 401 或 token revoked。刷新 OAuth 并重建 client。
- `ECONNRESET`/`EPIPE`。可关闭 keep-alive 后建立新连接。
- provider 返回 input + max_tokens 超 context limit。根据剩余空间降低 output budget。

每次等待前会 yield `SystemAPIErrorMessage`。TUI/SDK 因此可以显示“正在重试”状态。

### 不重试的情况

- 明确的 quota/billing/credit 耗尽。
- OpenCode Go 等带重置语义的订阅配额终止错误。
- 不可重试的兼容性分类。
- 用户或父任务 abort。
- 参数/认证错误在刷新后保持不可修复状态。

### 529 和容量风暴

后台标题、建议、摘要等非前台 query 默认不重试 529，避免服务过载时后台请求放大流量。前台主查询允许重试。连续 529 达到 3 次后可触发模型 fallback，外部用户无 fallback 时转为明确的 repeated-overload 错误。

Fast mode 遇到短 `Retry-After` 会保持同一模型等待，以保留 prompt cache。等待过长或未知时进入 cooldown 并切回标准速度。API 明确拒绝 fast mode 时也只降级一次。

### 无人值守持久重试

仅在编译 feature 和 `CLAUDE_CODE_UNATTENDED_RETRY` 同时开启时，429/529 可进入持久 retry：

- 最多 100 次。
- 退避封顶 5 分钟。服务端 reset 最长等待受 6 小时 cap 约束。
- 长等待每 30 秒 yield heartbeat，防止宿主判定任务空闲。

这是专用模式，普通交互会话不会无限等待。

## 12.4 Query 层恢复状态机

API retry 只处理同一请求的再次发送。上下文、模型或 token 参数发生变化时，`query.ts` 通过显式 transition 重启本轮。

| Transition | 状态变化 | 防循环条件 |
|---|---|---|
| `collapse_drain_retry` | 提交已 staged context collapse | 同一恢复链只 drain 一次 |
| `reactive_compact_retry` | 用新摘要上下文重试 | `hasAttemptedReactiveCompact` |
| `context_overflow_compact_retry` | 强制 auto compact 后重试 | 每轮一次 |
| `provider_max_tokens_retry` | 使用 provider 报告的 output cap | cap 必须更低，且只一次 |
| `provider_fallback_retry` | 切换 provider profile 和主模型 | 每轮一次 |
| `max_output_tokens_escalate` | 从默认小 cap 升到更高 cap | 仅未显式覆盖时 |
| `max_output_tokens_recovery` | 注入 continuation meta prompt | 最多 3 次 |

这些标志属于当前 query state。普通消息不持久化这些标志。持久化会使 resume 错误继承一次性重试状态。

### Context overflow

处理顺序为：

1. 若 context-collapse 有已 staged 段，先提交它们。
2. 可用时尝试 reactive compact。
3. 对标准 `context_overflow` assistant error，强制 auto compact 并重试一次。
4. 前述恢复失败时显示原错误并执行 StopFailure hooks。

当前源码定义能力边界。`src/services/compact/reactiveCompact.ts` 在此快照中是 feature-gated stub。`isReactiveCompactEnabled()` 恒为 `false`。当前开源构建未执行 reactive compact。可用路径包括标准 auto compact 和 context collapse。

API error 后不会运行普通 Stop hooks。模型没有产生有效回答时执行 Stop hook，容易形成“API error -> hook 阻止结束 -> 同一 API error”的循环。这里仅触发 StopFailure。

### Output token 上限

该错误表示 provider 已接受输入，并在输出达到上限后截断响应。context overflow 表示输入窗口不足。

- provider 报告硬上限时，解析数值并以更低 cap 重试一次。
- feature 开启且用户没有显式设 cap 时，可先从默认 cap 升到 `ESCALATED_MAX_TOKENS`。
- 响应继续被截断时保留已产生的 assistant 内容，并追加“直接继续”的 meta user message，最多 3 次。
- 达到上限后显示原错误，不再递归续写。

OpenRouter 风格的 HTTP 402 若提供“当前余额可承受的 max_tokens”，API retry 层只降额一次。第二次 402 直接失败。

### Provider fallback

rate limit 且配置了 `providerFallbackChain` 时，query 可激活下一 profile，并同时更新 endpoint 对应的主模型后重试一次。compact 和 session-memory fork 不做此切换，避免后台 fork 改写外层会话已经选择的 provider。

## 12.5 Auto Compact 熔断器

Auto compact 在正常阈值外还可由内存压力或消息数强制触发。若压缩本身持续失败，立即重试会产生额外 API 请求并继续增大上下文。

当前熔断规则：

- 连续失败 3 次后 open。
- 默认冷却 5 分钟。
- 环境覆盖不得低于 10 秒，非法值回退默认值。
- 冷却结束允许一次 half-open 尝试。
- 成功后清空失败计数。half-open 尝试失败时重新进入冷却。
- 用户手动 compact 不应被后台自动熔断状态无提示吞掉。

熔断状态由 query caller 线程化保存，不写入 transcript。恢复新进程时从干净状态开始。

## 12.6 Tool Failure Loop Guard

模型可能重复使用同一错误路径或相同无效参数。`toolFailureLoopGuard` 统计三种键：

- tool name + 归一化错误类别。
- 错误类别。
- 被修改文件路径。

默认阈值为 3，可由 `CLAUDE_CODE_TOOL_FAILURE_LOOP_THRESHOLD` 调整。`0` 禁用。阈值前一次会向模型加入 advisory，达到阈值则返回终止 assistant message。

成功执行会清理相关短期计数。某工具成功会清理该工具的持久 signature。文件 mutation 成功只清理对应路径。用户 abort、父 query 结束、background 等生成的合成 tool result 不计为工具失败。真正的 tool timeout 会计入。

该 guard 不把原始路径、工具输入或完整错误写入分析事件，只记录类别性字段，降低遥测泄漏风险。

## 12.7 Abort 的原因传播

Abort 携带类型化 reason。常见 reason 包括：

```text
user-abort, interrupt, query-timeout, hard-max-query-timeout,
background, side-task-cancelled, tool-timeout, parent-ended,
agent-summary-superseded, memory-extraction-superseded
```

父查询使用 child AbortController 向下传播取消信号。child abort 不会反向取消 parent。parent 监听器使用 WeakRef。被遗弃的 child 可以被垃圾回收。child 主动 abort 时会移除 parent listener。

组合 signal 同时监听父 signal、第二 signal 和 timeout，并返回显式 `cleanup()`。这里不直接依赖 Bun 的 `AbortSignal.timeout()`，因为其 native timer 在超时前可能积累内存。

不同 reason 决定 UI 和 transcript 行为：用户中断产生 interruption message。后台化或 timeout 产生相应 system warning。被新一代 side task 取代的旧摘要/记忆任务可以静默结束。

## 12.8 中断后的 Tool 配对修复

Anthropic 消息协议要求每个 `tool_use` 有对应 `tool_result`，ID 唯一且角色交替正确。流在两者之间中断、resume 从半轮开始或并行消息错误合并时，provider 会返回 400。

`ensureToolResultPairing()` 在发送 API 前执行防御性修复：

- 缺失 result：插入标记为 error 的合成 `tool_result`。
- 无对应 use 的 result：删除该 block。
- 重复 tool_use/result ID：只保留一个。
- 中断的 server-side `server_tool_use`/`mcp_tool_use`：删除无同消息 result 的 use。
- 删除后 assistant/user 内容为空：插入明确占位，保持合法角色序列。

普通产品运行优先恢复会话可用性。严格训练数据模式则在发现不配对时抛错，不允许合成内容污染轨迹。

恢复旧 transcript 时，`conversationRecovery.ts` 还会：

- 过滤未完成 tool use 之后的合成尾部。
- 清理孤立 thinking-only assistant message。
- 判断最后是未响应 user prompt、被工具中断的 turn，还是已完成 turn。
- 对真正中断的 turn 注入明确 continuation。正常结束于 tool result 的会话保持原状态。

例如 brief mode 中 `SendUserMessage` 的结果可合法终止一轮，恢复器会回溯对应 tool name，避免注入多余的“继续”。

## 12.9 流停滞和非流式降级

兼容 provider 可能建立连接后不再发送 chunk。stream reader 使用 idle timeout。每个有效 chunk 重置计时，真正停滞才 cancel reader。父 signal 已 abort 时保留父 abort 原因，不误报 idle timeout。

在允许的 provider 路径中，stream idle/协议失败可降级到非流式请求。`CLAUDE_CODE_DISABLE_NONSTREAMING_FALLBACK` 可关闭。一次性 guard 约束降级次数，并阻止 streaming 与 non-streaming 之间的反复切换。

Malformed JSON、HTML 网关错误页、缺少工具字段等会被兼容层分类为 provider response 问题，并显示 endpoint/proxy 检查建议。

## 12.10 局部失败隔离

| 故障 | 降级行为 |
|---|---|
| 一个 MCP server 连接失败 | 标记 failed/needs-auth，其他 server 可继续 |
| 一个 plugin 组件损坏 | 记录 PluginError，其他插件/组件继续 |
| 一个 LSP server 初始化失败 | 跳过该 server，保留其他扩展能力 |
| 非阻断 hook 失败 | 显示/记录错误，不终止主循环 |
| 大工具结果写盘失败 | 使用原始 inline result |
| 外部 settings 文件非法 | 保留错误，不把非法内容当有效配置 |
| 会话某行 JSON 损坏 | 跳过/隔离，不信任为消息 |
| cleanup 单项失败 | 聚合或忽略，继续清理其他资源 |

## 12.11 内存压力

Memory pressure monitor 默认每 30 秒检查 RSS，单会话预算默认取 `OPENCLAUDE_MAX_MEMORY_MB` 或 1,536 MiB：

- 80% 为 elevated。
- 90% 为 critical。
- critical 清理所有注册的可裁剪缓存。
- elevated/critical 持续设置一次性 compact request。消费后重置，由 auto-compact cooldown 防止风暴。

此外，大 tool result 外置、轻量 session list、compact-boundary chunked read 和消息虚拟化共同降低长会话内存。它们比事后捕获 OOM 更重要，因为 V8 进程在真正 OOM 时通常没有可靠恢复空间。

## 12.12 Graceful Shutdown

SIGINT、SIGTERM、SIGHUP、终端失联和显式 `/exit` 最终进入幂等 `gracefulShutdown()`：

1. 标记 shutdown，阻止重复进入。
2. 等一帧让 React 清除浮层。
3. 恢复终端模式并先打印 resume hint。
4. 运行注册 cleanup，优先确保 session flush。cleanup budget 约 2 秒。
5. 运行 SessionEnd hooks、关闭 MCP/子进程、刷新遥测等。
6. 全局 failsafe 到期时强制退出。预算至少 5 秒，并按 SessionEnd hook timeout 增长。

SIGTERM 使用退出码 143。SIGHUP 使用退出码 129。macOS TTY 被撤销且未收到 SIGHUP 时，系统每 30 秒检测一次 stdin/stdout 可用性并触发清理。

`uncaughtException` 和 `unhandledRejection` 会写无 PII 诊断与事件。业务恢复依赖显式 cleanup registry。

## 12.13 排障顺序

1. 先识别错误来自 provider、query、tool、hook、MCP 还是 UI。
2. 查看结构化 error/category，不先依赖显示文案。
3. 确认一次性 transition 状态，防止手动叠加重试。
4. 检查 abort reason 和 `AbortError` 名称。
5. API 400 优先验证 tool pair、上下文长度和 provider tool compatibility。
6. 长会话检查 compact breaker、工具结果外置和 memory pressure 日志。
7. 退出问题检查 cleanup 注册、session flush 和子进程信号升级。

下一章比较交互 CLI、print/headless、SDK、MCP server、gRPC 与远程入口的控制流差异。
