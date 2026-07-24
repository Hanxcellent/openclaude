# 13. 入口模式、SDK、远程控制与 gRPC

OpenClaude 的各入口共享同一 QueryEngine 和工具定义。每个入口具有独立的 UI、输入协议和权限交互。入口差异决定具体功能的可用性。

## 13.1 发布入口

npm 包公开两个主要运行产物：

| Export | 源入口 | 用途 |
|---|---|---|
| `bin/openclaude` -> `dist/cli.mjs` | `src/entrypoints/cli.tsx` | 交互 CLI、print、管理子命令、remote 等 |
| `@gitlawb/openclaude/sdk` -> `dist/sdk.mjs` | `src/entrypoints/sdk/index.ts` | Node 进程内 SDK |

`src/entrypoints/mcp.ts` 由 CLI 的 MCP 子命令调用。`src/grpc/` 通过 `scripts/start-grpc.ts` 启动。该入口属于仓库开发脚本。package exports 未公开 gRPC 入口。其接口不具备公开稳定性承诺。

## 13.2 CLI Bootstrap

`entrypoints/cli.tsx` 先处理低成本 fast path，再动态导入完整 `main.tsx`。主要顺序为：

```text
--version
  -> ps/logs/attach/kill
  -> provider env file / --provider
  -> enable config + managed env
  -> skills 管理 fast path
  -> provider profile + flag settings + agent route
  -> --background
  -> credential hydration + provider validation
  -> startup screen
  -> MCP/native host/remote-control/daemon 等特殊入口
  -> main.tsx Commander 完整解析
```

该顺序满足明确的启动约束。provider 参数必须在认证检查和启动画面前生效。skills 本地管理需要在 provider 配置损坏时保持可用。后台 child 必须继承已经解析的 provider/model 环境。

`--version` 不加载完整模块图，背景 session 的本地管理也不需要 API 认证。

## 13.3 交互 TUI 模式

默认入口在 TTY 中渲染 App/REPL：

- PromptInput 接收文本、paste、图片、slash command。
- React/AppState 持有对话和权限 UI 状态。
- MCP、LSP、插件、hooks、settings watcher 在启动期接入。
- 工具权限可通过 Ink dialog 异步询问。
- 输出通过普通或 alternate-screen renderer 展示。

这一路径拥有最完整的人机交互能力。任何返回 `local-jsx` 的命令都默认依赖此入口。

工作目录信任 gate 在执行项目 hooks、plugins 和 instructions 前完成。交互模式使用独立的信任确认流程。`--print` 由调用者承担目录信任责任。

## 13.4 Print/Headless 模式

`-p/--print` 调用 `src/cli/print.ts`，不挂载 React tree。它自己创建 AppState、权限回调、settings change subscriber、MCP 和 query 生命周期。

### 输出格式

| 格式 | 输出行为 | 典型用途 |
|---|---|---|
| `text` | 最终文本 | shell 管道 |
| `json` | 单个结果对象。verbose 可含完整消息 | 一次性机器消费 |
| `stream-json` | NDJSON 实时事件 | SDK host、远程控制、长任务 |

`stream-json` 要求 `--verbose`。`--include-partial-messages` 可发出增量 assistant chunk。`--include-hook-events` 增加 hook lifecycle event。`--json-schema` 要求最终结构化输出通过指定 schema。

### 输入格式

- `text`：命令行 prompt 或 stdin 文本。
- `stream-json`：`StructuredIO` 按行解析 SDK user/control message，可连续提交多轮。

`--replay-user-messages` 只在 stream-json 双向协议中用于 acknowledgment。stdout 必须保持机器可解析，诊断、startup heartbeat 等应走 stderr 或结构化事件。

### Headless 特有限制

- 必须有 prompt/stdin，除非由 orchestrator 后续注入。
- 没有 Ink permission dialog。应使用 allow/deny rules、SDK control permission response 或 `--permission-prompt-tool`。
- `--max-turns`、`--max-budget-usd` 提供非交互终止条件。
- `--no-session-persistence` 只适用于 print。
- print 跳过 workspace trust dialog，因此 CLI 文案要求只在可信目录使用。
- `--heartbeat` 只适用于 print，输出安静时提供 liveness。

### Bare 模式

`--bare` 设置简化模式。该模式跳过 hooks、LSP、插件同步、attribution、auto-memory、后台预取、keychain 和自动 CLAUDE.md 发现。显式 MCP、settings、agents、plugin-dir、add-dir 和 skill 保持可加载状态。

Anthropic 认证在 bare 下只读取显式 API key 或 settings 中的 apiKeyHelper，不读取 OAuth/keychain。第三方 provider 继续使用自己的凭据。

## 13.5 StructuredIO 与 RemoteIO

`StructuredIO` 把 stdin NDJSON 转成：

- 用户消息。
- interrupt、permission response、set model/mode 等 control request。
- MCP elicitation response。

同时负责 control response 的相关 ID、输入回放和错误格式。

存在 `--sdk-url` 时使用 `RemoteIO`。它在 StructuredIO 之上连接远端 session transport，并维护 internal-event queue、断线恢复和远端确认。业务 query 在本地 headless engine 执行。输入输出由远端控制平面传输。

Transport 实现包括 WebSocket、SSE 和 hybrid。Hybrid 可根据服务能力或连接状态选择通道，并统一向 `StdoutMessage` 事件模型转换。

## 13.6 SDK v1 Query

SDK barrel 禁止导入 React、Ink 或 CLI/TUI 代码。构建过程检查被错误替换成 stub 的关键模块。

`query()` 返回同时实现 `AsyncIterable<SDKMessage>` 和控制方法的 `Query`：

- 迭代时才完成 init、agent 装载、MCP 连接与 resume 解析。
- prompt 可以是字符串，也可以是异步 SDK user message 流。
- 支持 interrupt、close、setModel、setPermissionMode、respondToPermission。
- 可读取当前 messages、MCP status、支持的 agents/commands。
- 可检查并异步执行 file rewind。

`queryAsync()` 是需要先完成异步创建的对应形式。具体选择应以公开类型和调用要求为准。

### SDK 上下文隔离

历史 CLI 代码使用部分全局/模块状态。SDK 用 `runWithSdkContext()` 为当前异步链提供 session ID、cwd 和 transcript dir，并对 process env 覆盖使用全程 mutex：

1. 获取 mutex。
2. 保存被覆盖 env。
3. 执行 query。
4. `finally` 恢复 env 并释放 mutex。

这防止同进程并发 SDK query 相互覆盖 provider credentials。大量带不同 env override 的 query 会被串行化。SDK session 共享部分进程级资源。

Agent definition 或 MCP 装载失败会产生 SDK failure event 或警告。初始化过程继续加载剩余能力。

### SDK resume/fork

SDK 解析 `continue`、session ID、fork 和 `resumeSessionAt`：

- continue 查当前 cwd 最新会话。
- resume 从 JSONL 父链注入历史。
- fork 先生成新会话，再注入新文件的历史。
- 最终用解析出的 transcript dir 和 session ID 切换写入目标。

它复用 compact-aware transcript loader，不能把 JSONL 简单读成数组后全部注入。

## 13.7 SDK V2 持久会话

V2 API 当前带 `unstable_` 前缀：

```text
unstable_v2_createSession(options)
unstable_v2_resumeSession(sessionId, options)
unstable_v2_prompt(message, options)
```

`SDKSession` 持有一个长期 QueryEngine，可多次 `sendMessage()`。`interrupt()` 只中止当前 query。`close()` 还释放 MCP client、permission pending map、timeout/failure queues 和 engine 引用。

长生命周期 host 必须在 `finally` 调用 `close()`。缺少 `close()` 会使 abandoned session 保留 buffer 和 pending callbacks。

V2 权限是 secure-by-default：没有 `canUseTool` 或 `onPermissionRequest` 时，不能假定工具会自动允许。外部 permission request 默认有约 30 秒响应窗口。host 用 tool-use ID 调用 `respondToPermission()`。

`unstable_v2_prompt()` 是一次性封装，内部创建 session、收集 result，并在 `finally` 自动 close。

### SDK 内嵌 MCP

`tool()` 构造 in-process MCP tool definition。`createSdkMcpServer()` 只给配置加 `scope: session`，本身不启动 server。session 迭代开始后才按 stdio/SSE/HTTP/sdk 类型连接并把工具合并进 pool。

## 13.8 Session Management SDK

SDK 还提供不启动模型的轻量操作：

- `listSessions()`、`getSessionInfo()`。
- `getSessionMessages()`。
- `renameSession()`、`tagSession()`、`deleteSession()`。
- `forkSession()`。

所有传入 session ID 先校验 UUID。列表和 info 使用轻量头尾读取。messages 沿 parent UUID 选择活动链并排除 sidechain。Mutation 直接写 JSONL 元数据，因此调用者需要处理与正在运行会话的并发写入。

## 13.9 MCP Server 入口

`startMCPServer(cwd, debug, verbose)` 让 OpenClaude 自身成为 stdio MCP server，只暴露 `tools/list` 与 `tools/call`。

能力来源为内建工具加当前配置的外部 MCP tools。若重名，外部 MCP tool 优先，内建项被去重。输出 schema 只有根为 object 时才暴露，避免 MCP SDK 不接受 union 根。

调用路径比 REPL 更薄：

1. 查找且检查 tool enabled。
2. Zod 解析 input。
3. 运行 tool 自身 `validateInput()`。
4. 直接调用 `tool.call()`，传入非交互 context 和权限 helper。
5. 映射 text/image/error 到 MCP result。

该入口不提供对话历史、Ink 权限 UI、完整 slash command 或 agent definitions。入口仅保留 review command。其 Hook 和 UI 行为范围以当前调用路径为准。

## 13.10 Remote、Assistant 与 SSH

仓库存在三类远程概念：

| 模式 | 执行位置 | 本地 TUI 的角色 |
|---|---|---|
| `remote-control` bridge | 本机执行工具 | 向移动端/web 暴露会话并转发事件 |
| `assistant [sessionId]` | bridge session 已存在 | 作为客户端附着、查看/输入 |
| remote session | 远端 agent 执行 | 本地显示服务端消息，不重复执行工具 |
| SSH mode | SSH 目标执行 CLI/agent | 本地转发流，断线有限重连 |

Remote Control 启动前检查登录、版本、feature gate 和组织策略。Bridge 输入的 slash command 还经过第 11 章的远程安全过滤。

`useRemoteSession` 收到远端已经生成的 assistant/tool 状态时，仅更新本地显示。本地重新执行工具会造成双重副作用。SSH 断线会显示重连状态。达到最大次数后进入 graceful shutdown。

远程模式共享显示协议。各执行端保有独立的 filesystem、MCP 和 permission context。工具执行位置由该模式的 transport 和 QueryEngine 所在进程决定。

## 13.11 后台 CLI 会话

`--bg/--background` 派生新的 `--print` child，将进程身份、session ID、命令和日志位置写入 registry。`ps/logs/attach/kill` 走轻量 fast path。

kill 前会重新验证 PID 和进程身份。系统先发送 SIGTERM，再按平台和存活状态升级。该验证防止 PID 复用导致误杀进程。后台 agent/task 与主会话内的 in-process subagent 使用独立机制，详见第 8 章。

## 13.12 gRPC 开发适配器

`src/grpc/server.ts` 提供双向 `AgentService.Chat`：client 发送 request、permission input 或 cancel。server 发送 text chunk、tool start/result、action required、done/error。

实现特征：

- 每条 stream 同时只允许一个 request。
- 每个 request 创建 QueryEngine，并注册 built-in agents。
- 工具使用前发 `action_required`，客户端以 prompt ID 回答 yes/no。
- cancel 中断 engine 并结束 stream。
- session history 仅保存在 server 进程的 `Map`，最多 1,000 个，按插入顺序淘汰。
- server 使用 `createInsecure()`，默认只适合 localhost 开发。

该实现缺少生产级持久化、认证、TLS、多租户隔离、权限超时和分布式 session store。`scripts/start-grpc.ts` 还包含开发用 MACRO polyfill。准确的项目表述是“仓库包含 QueryEngine 的 gRPC 原型适配器”。

## 13.13 入口能力矩阵

| 能力 | TUI | Print | SDK | MCP server | gRPC 原型 |
|---|---:|---:|---:|---:|---:|
| 多轮 | 是 | stream-json 可多轮 | 是 | 否 | 是 |
| Ink UI | 是 | 否 | 否 | 否 | 否 |
| 人工权限 | dialog | control/tool | callback/event | host 决定 | action_required |
| 会话 JSONL | 是 | 可选 | 是 | 否 | 仅内存 Map |
| MCP client | 是 | 是 | 可配置 | 可转暴露 | 当前无 |
| Plugin/LSP | 完整 | 依模式 | 非 TUI 子集 | 否/有限 | 否 |
| 公开稳定性 | CLI | CLI | v1 公开、V2 unstable | CLI 子入口 | 开发脚本 |

下一章按照攻击面审视信任、权限、进程、网络和数据边界。
