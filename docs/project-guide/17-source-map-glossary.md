# 17. 源码导航、术语表与学习路线

本章提供按问题定位源码的索引。源码持续演进。检索应优先使用导出符号和数据类型。固定行号仅表示当前快照位置。

## 17.1 最小源码主干

第一次只读以下路径，就能建立端到端骨架：

```text
bin/openclaude
  -> src/entrypoints/cli.tsx
  -> src/entrypoints/init.ts
  -> src/main.tsx
  -> src/components/App.tsx
  -> src/utils/processUserInput/
  -> src/query.ts + src/QueryEngine.ts
  -> src/services/api/claude.ts
  -> src/tools.ts + src/Tool.ts
  -> src/utils/permissions/
  -> src/utils/log.ts + src/utils/sessionStorage.ts
```

阅读顺序应从这条主干开始，再进入 provider、MCP、Agent 或 UI 子系统。该顺序有助于保持完整控制流。

## 17.2 顶层目录职责

| 目录 | 主要职责 | 进入时机 |
|---|---|---|
| `src/entrypoints/` | CLI、init、MCP、SDK 和公开类型 | 入口、参数、宿主集成 |
| `src/components/` | React/Ink 页面、消息、Dialog | 终端显示和交互 |
| `src/hooks/` | TUI 行为组合、队列、取消、插件管理 | UI 到状态/服务的桥 |
| `src/state/` | AppState store、selector、变更副作用 | 共享可订阅状态 |
| `src/context/` | React context、modal、mailbox、通知 | 局部 UI ownership |
| `src/query/` | query 辅助状态机、工具失败保护 | Agent loop 深入逻辑 |
| `src/services/api/` | provider client、请求/stream/error 适配 | 多模型和网络协议 |
| `src/tools/` | 内置与包装工具实现 | 模型可执行能力 |
| `src/utils/permissions/` | permission mode/rule/path/safety | 自动批准与拒绝 |
| `src/utils/settings/` | 配置源、schema、merge、policy | 配置优先级和治理 |
| `src/services/mcp/` | MCP transport、discovery、auth | 外部 tool/resource |
| `src/utils/plugins/` | marketplace、加载、安装、依赖 | 扩展分发和信任 |
| `src/utils/hooks/` | Hook 匹配、进程/HTTP 执行、输出 | 生命周期拦截 |
| `src/services/lsp/` | LSP client/server manager | 代码智能插件 |
| `src/tasks/` | shell、agent、teammate、workflow、remote task | 长任务生命周期 |
| `src/utils/swarm/` | team、mailbox、计划审批、协作 todo | 多 Agent 团队 |
| `src/remote/`、`src/bridge/` | WebSocket/remote control/ingress | 远程会话 |
| `src/ssh/` | SSH 启动、转发和远程环境 | `openclaude ssh` |
| `src/memdir/` | session/team memory 路径和索引 | 长期记忆 |
| `src/integrations/` | provider/model descriptor 与生成元数据 | 增加集成 |
| `src/ink/` | 自维护终端 renderer | 底层渲染/输入问题 |
| `src/grpc/` | 开发用 gRPC adapter | 非生产 RPC 实验 |
| `scripts/` | build、生成、诊断、发布守卫 | 工程/构建 |

`daemon`、`proactive`、`environment-runner`、`self-hosted-runner` 等目录具有独立的构建状态。当前可用性需要对照 `scripts/build.ts` 的 feature flags 和 stub。

### 17.2.1 其余顶层目录与当前状态

以下目录位于主循环外围，阅读仓库时会遇到：

| 目录 | 职责 | 当前 open build 的判断 |
|---|---|---|
| `src/assistant/` | KAIROS 持久 assistant 的 session 选择与历史 | `KAIROS=false`，entitlement gate 恒 false |
| `src/buddy/` | 终端像素伙伴、动作效果和 shot clock 观察 | `BUDDY=true`。属于 UI/反馈，不改变 query 语义 |
| `src/coordinator/` | coordinator prompt、worker tools 和 session mode | 代码已构建。还需 `CLAUDE_CODE_COORDINATOR_MODE` 运行时开启 |
| `src/daemon/` | daemon supervisor/worker registry | `DAEMON=false`，入口为 inert/stub |
| `src/environment-runner/` | BYOC headless runner | 当前为 inert stub |
| `src/i18n/` | locale 检测和命令文案字典 | 当前提供英文、越南文与 fallback |
| `src/jobs/` | template/job-directory turn classifier | 当前实现明确为 inert stub |
| `src/migrations/` | 启动时迁移旧设置、模型名和开关 | live startup compatibility。范围限于配置迁移 |
| `src/native-ts/` | 原生能力的 TypeScript 替代，如 fuzzy file index、Yoga/color helpers | open build 降低 native addon 依赖 |
| `src/outputStyles/` | 从 Markdown/frontmatter 加载输出风格 prompt | 与 plugin output style cache 合并 |
| `src/plugins/` | 内置插件注册表和 bundled plugin 初始化 | 与 `src/utils/plugins/` 的 marketplace/安装实现分工 |
| `src/proactive/` | proactive active/pause/tick 状态 | `PROACTIVE=false`，主入口分支被裁剪 |
| `src/proto/` | `openclaude.proto` 双向 Chat stream 契约 | 供开发 gRPC adapter 使用。npm package exports 未公开 |
| `src/self-hosted-runner/` | self-hosted worker register/poll | 当前为 inert stub |
| `src/skills/` | bundled skill registry、目录加载和 MCP skill builder | 多个 bundled skills live。各 skill 可以具有独立 gate |
| `src/upstreamproxy/` | CCR remote container 的本地 relay、CA 和子进程代理注入 | 仅 `CLAUDE_CODE_REMOTE` 等受控环境条件满足时启用 |
| `src/vim/` | Vim motion、operator、text object 和 transition | TUI 输入模式子系统 |
| `src/voice/` | voice feature/auth 可见性判断 | `VOICE_MODE=false`，当前产物不可用 |

`src/types/`、`src/schemas/` 和 `src/constants/` 是横切定义层。`src/native-ts/` 提供替代实现。`src/__tests__/` 放置跨模块回归测试。`src/test/` 放置共享测试工具。

## 17.3 按主题定位源码

### 启动与参数

| 定位主题 | 第一站 | 后续 |
|---|---|---|
| 安装命令启动失败 | `bin/openclaude` | `package.json` engines/exports |
| 参数定义 | `src/entrypoints/cli.tsx` | `src/main.tsx` action handler |
| 启动初始化顺序 | `src/entrypoints/init.ts` | `src/setup.ts`、`src/interactiveHelpers.tsx` |
| print 与 TUI 分流 | `src/main.tsx` | `src/cli/print.ts` |
| 退出清理失败 | `src/utils/gracefulShutdown.ts` | `src/utils/cleanupRegistry.ts` |

### 输入与命令

| 定位主题 | 第一站 | 后续 |
|---|---|---|
| Enter 提交流程 | PromptInput/REPL submit 路径 | `src/utils/processUserInput/` |
| slash command 注册 | `src/commands.ts` | `src/commands/<name>/` |
| skill/command 加载 | `src/utils/skills/` | plugin command loader |
| `!cmd` 执行 | `processBashCommand.tsx` | BashTool/LocalShellTask |
| 运行中输入处理 | `src/hooks/useCommandQueue.ts` | QueryGuard/queue processor |

### Agent loop

| 定位主题 | 第一站 | 后续 |
|---|---|---|
| 一轮模型请求 | `src/query.ts` | `src/QueryEngine.ts` |
| stream 消息转换 | `src/services/api/claude.ts` | provider adapter |
| 工具执行时机 | query 的 tool-use branch | `src/tools.ts`/tool execution helpers |
| Stop hook 运行时机 | query terminal branch | Hook service |
| Abort 传播失败 | query AbortSignal | runTools → tool.call → child process |
| 失败循环 | `src/query/toolFailureLoopGuard.ts` | retry/transition flags |

### Context 与 prompt

| 定位主题 | 第一站 | 后续 |
|---|---|---|
| system prompt 分段 | `src/utils/systemPrompt.ts`、`src/constants/systemPromptSections.ts` | tool prompts |
| CLAUDE/OPENCLAUDE 指令注入 | `src/context.ts` | file discovery/settings |
| attachment 注入时机 | attachment collection helpers | query context build |
| token 估算 | token counting/context helpers | provider usage |
| compact 触发 | `src/services/compact/` | query recovery |
| context collapse 定义 | `src/services/contextCollapse/` | build feature |
| memory 存储位置 | `src/memdir/` | SessionMemory/extractMemories |

### Provider 与模型

| 定位主题 | 第一站 | 后续 |
|---|---|---|
| 当前 provider 来源 | provider profile utilities | settings/env/CLI precedence |
| model 能力描述 | `src/integrations/` | generated artifacts |
| Anthropic 请求 | `src/services/api/claude.ts` | auth/client factory |
| OpenAI-compatible 请求 | `src/services/api/openaiShim.ts` | stream adapter/errors |
| Gemini/Vertex 请求 | `src/services/api/geminiVertexClient.ts` | Google auth |
| Bedrock 请求 | API client factory | AWS credential helpers |
| 错误重试 | `src/services/api/errors.ts`、retry helper | query transition |
| provider fallback | provider profile/fallback helpers | query recovery |

新增 provider 时应先阅读 `docs/integrations/overview.md` 和对应 `docs/integrations/how-to/`。实现过程应从 descriptor 和协议差异分析开始。

### 工具与权限

| 定位主题 | 第一站 | 后续 |
|---|---|---|
| 工具可见性 | `src/tools.ts` | feature/model/mode filter |
| 参数校验失败 | Tool `inputSchema` | provider tool JSON parsing |
| 权限框触发 | `src/hooks/useCanUseTool.tsx` | tool `checkPermissions` |
| allow/deny 来源 | `src/utils/permissions/permissionsLoader.ts` | settings sources |
| 路径拒绝 | `pathValidation.ts` | `filesystem.ts`/fsOperations |
| Bash 只读判定 | `src/utils/bash/` | `src/utils/shell/` |
| sandbox 生效状态 | `sandbox-adapter.ts` | `/sandbox`/doctor |
| 大输出存储 | `src/utils/toolResultStorage.ts` | transcript/UI preview |

### Agent、Task 与团队

| 定位主题 | 第一站 | 后续 |
|---|---|---|
| Agent 定义合并 | AgentTool/agent definition loaders | plugins/settings |
| 子 Agent 运行 | `src/tools/AgentTool/` | `runAgent`/query fork |
| background 返回 | `src/tasks/LocalAgentTask/` | task notification queue |
| shell task 停止 | `src/tasks/LocalShellTask/` | `stopTask.ts` |
| teammate 与 subagent 差别 | `InProcessTeammateTask` | `src/utils/swarm/` |
| 跨 Agent 消息 | `SendMessageTool` | team mailbox |
| 协作任务依赖 | TaskCreate/Update/List tools | todo DAG utilities |
| worktree 隔离 | Enter/ExitWorktreeTool | Git worktree utils |

### UI 与终端

| 定位主题 | 第一站 | 后续 |
|---|---|---|
| 主界面组成 | `src/components/App.tsx` | REPL/input/messages |
| 状态订阅 | `src/state/` | selector/hooks |
| ESC 取消顺序 | `src/hooks/useCancelRequest.ts` | modal/keybinding context |
| Message 渲染 | `Messages.tsx`/`Message.tsx` | content block components |
| 滚动跳动 | virtual message/fullscreen paths | custom Ink renderer |
| CJK 宽度错误 | `src/ink/` line width/bidi | terminal size hooks |
| keybinding 覆盖 | `src/keybindings/` | settings/keybinding warnings |

### 配置与会话

| 定位主题 | 第一站 | 后续 |
|---|---|---|
| 配置优先级 | `src/utils/settings/settings.ts` | constants/types/schema |
| 环境变量应用时机 | `src/utils/managedEnv.ts` | trust/startup |
| JSONL 写入位置 | `src/utils/log.ts` | sessionStorage |
| resume 活动链 | resume/log selector | transcript loaders |
| rewind 作用 | `src/commands/rewind/` | file history snapshot |
| branch 作用 | `src/commands/branch/` | parent UUID/session IDs |
| compact boundary 保存 | compact service/log events | resume projection |

### 扩展系统

| 定位主题 | 第一站 | 后续 |
|---|---|---|
| MCP server 连接失败 | `src/services/mcp/` | config/auth/transport |
| MCP tool 包装 | `src/tools/MCPTool/` | tool discovery |
| plugin 发现 | `src/utils/plugins/` | startup checks/cache |
| plugin 依赖拒绝 | dependencyResolver | marketplace policy |
| Hook 工具阻断 | `src/utils/hooks/` | Hook output schema |
| HTTP Hook SSRF | `src/utils/hooks/ssrfGuard.ts` | proxy/sandbox network |
| LSP server 生命周期 | `src/services/lsp/manager.ts` | plugin integration |

### SDK 与远程

| 定位主题 | 第一站 | 后续 |
|---|---|---|
| SDK 公开 API | `src/entrypoints/sdk.d.ts` | `sdk/index.ts` |
| `query()` 核心复用 | `sdk/query.ts` | env mutex/permission adapter |
| V2 session | `sdk/v2.ts` | sessions/transcript |
| SDK permission | `sdk/permissions.ts` | external callback |
| Remote WebSocket | `src/remote/SessionsWebSocket.ts` | RemoteSessionManager |
| bridge ingress | `src/bridge/` | session ingress auth |
| SSH | `src/ssh/` | remoteIO/unix socket auth |
| gRPC | `src/grpc/server.ts` | `scripts/start-grpc.ts` |

## 17.4 关键数据类型

### Message

内部对话事件的共同语义，通常包含 role/type、content blocks、UUID、parent UUID、session metadata 和可选 usage/error。Assistant content 可以同时含 text、thinking 和 tool use。

### Content block

一条消息内部的结构单元。常见类型包括 text、thinking、tool_use、tool_result、image/document。assistant message 可以包含多个结构块。

### Tool

具备 name、description/prompt、input schema、permission check、call、并发/只读属性和 UI/result 映射的 capability descriptor。

### Tool use / Tool result

模型提出的能力调用与本地返回。通过 ID 一一配对，是 provider 接受历史和 Agent loop 正确性的核心不变量。

### Query

一次从现有消息开始的异步执行。一次 query 可以包含多个 model step、tool step 和 HTTP 请求。

### Turn / Step

Turn 更接近用户发起的一轮交互。step 常表示一次模型调用到下一次模型调用之间的推进。具体限制代码需按使用点确认，不能把所有计数统称“轮数”。

### AppState

TUI 和业务层可订阅的当前状态投影。transcript 和模块级资源由各自存储管理。

### Bootstrap state

初始化早期建立、供非 React 模块读取的进程/session 数据，如原始 cwd、session ID、非交互标志和 session trust。

### QueryGuard

主 query 的所有权与并发协调对象，防止多个 UI 触发器同时推进同一个主会话。

### Command queue

将用户 follow-up、steer、task notification、remote message 等按安全点送入主循环的队列。队列内容和 transcript 由独立结构管理。

### Provider descriptor

描述 provider/model 能力和默认值的元数据。不持有某次会话的凭据或网络 client。

### Provider profile

用户可选择/持久化的一组 provider、endpoint、model 和兼容设置。profile 解析后才构造 transport。

### Transport adapter

把内部消息/tools/options 转成供应商请求，并把 stream/error/usage 转回内部语义的协议层。

### Permission mode

缺少具体规则时的交互/自动审批策略，不等同 OS 权限，也不等同 sandbox。

### Permission rule

带来源、行为和 tool/specifier 的持久或会话授权条目。deny 通常高于 allow。

### Workspace trust

用户允许当前项目级配置进入执行链的确认。它不自动批准所有工具，也不证明项目无恶意代码。

### Sandbox

可选的 OS 进程文件/网络隔离层，主要约束 Bash 等子进程。启用状态、可用状态和强制状态是三个独立属性。

### Hook

在 Session/Prompt/Tool/Stop 等生命周期点运行的 command、HTTP 或 prompt callback，可阻止、修改、反馈或附加上下文。

### MCP

Model Context Protocol。外部 server 通过 transport 暴露 tools/resources/prompts 等 capability。OpenClaude 作为 client 时将其包装进本地执行链。

### Plugin

扩展的安装和分发单元，可以贡献 commands、agents、skills、hooks、MCP/LSP 配置。插件复用现有 Agent runtime。

### Skill

按需发现和加载的领域指令、资源和允许工具声明。其文本属于上下文，执行能力必须走工具边界。

### Task

长生命周期执行单元的统一记录，包含 ID、type、status、progress/output 和 stop 行为。Agent、shell、workflow、remote 可有不同实现。

### Subagent

从父会话 fork 局部上下文运行的 Agent。可同步或后台，不必是独立进程。

### Teammate

拥有 team identity、mailbox、idle/active 协议和协作任务的成员，比普通 subagent 有更长期的团队语义。

### Worktree

Git 层面的独立工作目录，用于并行修改隔离。与会话消息 branch 无关。

### Conversation branch

从某条历史消息建立新的 parent chain/leaf。共享当前文件系统，不创建 Git branch。

### Compact

把旧上下文摘要成更短的 prompt projection，同时用 boundary/event 保留 transcript 可恢复性。

### Context collapse

对上下文片段做 staged collapsing 的优化路径。它与标准摘要 compact 使用独立实现。

### Reactive compact

错误后尝试响应式压缩的概念。当前源码快照的实现是禁用 stub，不能按已启用能力理解。

## 17.5 容易混淆的概念对照

| 概念 A | 概念 B | 区别 |
|---|---|---|
| model | provider | 模型能力标识 vs 提供 API/认证/协议的后端 |
| profile | transport client | 持久选择配置 vs 某次运行构造的网络对象 |
| query | API request | 可含多步工具循环 vs 单次网络请求 |
| message | stream delta | 完整逻辑事件 vs 尚未完成的增量 |
| permission | sandbox | 应用层授权决策 vs OS 执行约束 |
| trust | permission | 启用项目控制面 vs 允许具体操作 |
| Hook | tool | 生命周期拦截器 vs 模型可调用 capability |
| Skill | plugin | 指令资源单元 vs 安装/分发能力包 |
| MCP client | MCP server entry | 连接外部能力 vs 把本地工具重暴露出去 |
| task | tool use | 长生命周期记录 vs 单次模型协议调用 |
| subagent | teammate | 局部 fork 执行 vs 有团队身份的协作者 |
| background | detached process | 生命周期不阻塞父 query vs 不一定是 OS 独立进程 |
| resume | branch | 继续已有 leaf vs 从历史点创建新 leaf |
| rewind conversation | rewind files | 改消息链 vs 恢复文件快照 |
| conversation branch | Git branch/worktree | 对话 parent chain vs 文件版本隔离 |
| compact | truncate | 语义摘要+boundary vs 机械删除 |
| retry | fallback | 同一路由重发 vs 改 provider/model/profile |
| context overflow | output truncation | 输入窗口不足 vs 回答达到输出上限 |
| feature gate | runtime setting | 构建时裁剪 vs 进程运行时选择 |
| telemetry | product network | 产品分析流量 vs 模型/MCP/WebFetch 等显式功能流量 |

## 17.6 推荐学习路线

### 阶段一：建立骨架

阅读 00-04 章，并亲自追踪一次无工具请求、一次 Read 请求和一次 Bash 被拒绝请求。完成标准：能画出主循环并解释 tool pair。

### 阶段二：理解模型与上下文

阅读 05-06 章，对比一个 Anthropic 与一个 OpenAI-compatible request/stream。完成标准：能够区分 descriptor 差异与 adapter 必须处理的差异。

### 阶段三：理解能力与安全

阅读 07、11、14 章，手工跟踪 FileEdit、Bash 和 MCP tool 的 permission。完成标准：能说清 trust、schema、Hook、permission、safety、sandbox 的顺序。

### 阶段四：理解并发状态

阅读 08-09 章，跟踪 background agent 完成通知和运行中 follow-up。完成标准：能区分 QueryGuard、command queue、task registry 和 AppState。

### 阶段五：理解恢复与入口

阅读 10、12、13 章，检查 resume、compact、abort、headless 和 SDK。完成标准包括共享核心的协议一致性和各入口的专用适配。

### 阶段六：形成工程判断

阅读 15-16 章，选择一个真实改动，写出风险、不变量、focused tests 和完整 PR 验证。完成标准包括功能、权衡、失败模式和限制的完整说明。

## 17.7 实践练习

1. 给一次 query stream 画出所有 yield 类型，并标注持久化类型。
2. 构造两个并发只读 tool use 和一个 FileEdit，预测调度顺序。
3. 找出 project settings 试图关闭 sandbox 时的判定路径。
4. 构造 symlink 指向工作区外，跟踪 FileRead 与 FileEdit 的差异。
5. 对比 OpenAI-compatible 与 Anthropic 的 tool result 请求形态。
6. 从 JSONL 某个 leaf 手工沿 parent UUID 还原历史。
7. 跟踪 background Agent 从 tool return 到 task notification。
8. 在 SDK 中实现最小 `canUseTool`，确保只 resolve 一次并处理 abort。
9. 模拟 context overflow，列出 API retry 和 query transition 的边界。
10. 修改一个 feature flag，并预测源码、bundle 和 smoke test 的变化。

## 17.8 完整性检查矩阵

读完全部章节后，应满足以下标准：

| 领域 | 验收标准 | 对应章节 |
|---|---|---|
| 架构 | 五层协作和控制面/数据面分离 | 00 |
| 构建 | feature、stub、CLI/SDK bundle 形成过程 | 01、15 |
| 启动 | TUI/headless/SDK/MCP 分流 | 02、13 |
| 状态 | 四类状态和消息协议一致性 | 03、09 |
| 主循环 | 输入到 tool result 再到终止的全时序 | 04 |
| 上下文 | prompt、memory、attachment、compact | 05 |
| Provider | 配置、路由、协议、错误和 usage | 06 |
| 工具安全 | schema、Hook、permission、path、sandbox | 07、14 |
| 多 Agent | sync/background/team/task/worktree | 08 |
| 持久化 | JSONL、parent chain、resume、branch | 10 |
| 扩展 | command/plugin/MCP/Hook/LSP | 11 |
| 异常 | retry、overflow、abort、loop guard、shutdown | 12 |
| 多入口 | print、structured IO、SDK、remote、gRPC | 13 |
| 工程 | tests、CI、调试、性能和贡献约束 | 15 |
| 表达 | 分层讲解、追问、限制和个人贡献 | 16 |

只掌握功能名称的领域需要继续回到对应源码完成具体状态迁移追踪。

## 17.9 快速检索命令

```bash
# 找类型/符号定义
rg -n "export (type|interface|class|function|const) .*Name" src

# 找调用点
rg -n "Name\(" src

# 找构建期开关
rg -n "feature\('[A-Z0-9_]+" src scripts/build.ts

# 找某设置的所有来源和 schema
rg -n "settingName" src/utils/settings src/entrypoints/sandboxTypes.ts

# 找某工具的 prompt/schema/permission/call
rg -n "name:|inputSchema|checkPermissions|call\(" src/tools/<ToolName>

# 找关联测试
rg --files src scripts | rg 'subject.*test\.(ts|tsx)$'
```

搜索用于建立候选调用图。live path 判断还需要检查 feature gate、dynamic import、入口参数和 build stub。

## 17.10 指南索引

- [00 全景与核心心智模型](00-architecture-overview.md)
- [01 仓库、构建与运行时](01-repository-build-runtime.md)
- [02 入口与启动链路](02-entrypoints-startup.md)
- [03 状态与数据模型](03-state-and-data-model.md)
- [04 主 Agent 执行循环](04-query-agent-loop.md)
- [05 上下文、Prompt、记忆与压缩](05-context-prompt-memory.md)
- [06 模型供应商与协议适配](06-provider-model-transport.md)
- [07 工具、权限、沙箱与文件安全](07-tools-permissions-security.md)
- [08 Agent、任务、团队与编排](08-agents-tasks-orchestration.md)
- [09 TUI 与交互状态流](09-tui-interaction-flow.md)
- [10 配置与会话持久化](10-configuration-session-persistence.md)
- [11 MCP、插件、Hooks 与 LSP](11-mcp-plugins-hooks-lsp-commands.md)
- [12 错误恢复与特殊场景](12-errors-retries-recovery-edge-cases.md)
- [13 多入口与部署形态](13-entrypoint-modes-sdk-remote.md)
- [14 安全边界与威胁模型](14-security-model.md)
- [15 工程质量与测试方法](15-engineering-and-testing.md)
- [16 简历与面试讲解稿](16-interview-playbook.md)
