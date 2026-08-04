# 17. 附录：源码索引与术语

本附录面向需要查证实现细节的读者。主报告不依赖本章内容。本章提供按主题定位源码的索引和术语说明。源码持续演进，检索应优先使用导出符号和数据类型。

## 17.1 主流程源码路径

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

建议先按上述顺序阅读一次完整的启动、查询、工具执行和日志写入流程，再进入 provider、MCP、Agent 或 UI 子系统。

## 17.2 顶层目录职责

| 目录 | 主要职责 | 进入时机 |
|---|---|---|
| `src/entrypoints/` | CLI、init、MCP、SDK 和公开类型 | 入口、参数、调用程序集成 |
| `src/components/` | React/Ink 页面、消息、Dialog | 终端显示和交互 |
| `src/hooks/` | TUI 行为组合、队列、取消、插件管理 | UI 到状态/服务的桥 |
| `src/state/` | AppState store、selector、状态变化后的更新操作 | 共享可订阅状态 |
| `src/context/` | React context、modal、mailbox、通知 | 局部 UI 状态管理 |
| `src/query/` | query 状态处理、工具失败保护 | Agent loop 详细实现 |
| `src/services/api/` | provider client、请求/stream/error 适配 | 多模型和网络协议 |
| `src/tools/` | 内置与包装工具实现 | 模型可执行能力 |
| `src/utils/permissions/` | permission mode/rule/path/safety | 自动批准与拒绝 |
| `src/utils/settings/` | 配置源、schema、merge、policy | 配置优先级和治理 |
| `src/services/mcp/` | MCP 连接、能力发现和认证 | 外部 tool/resource |
| `src/utils/plugins/` | marketplace、加载、安装、依赖 | 扩展分发和信任 |
| `src/utils/hooks/` | Hook 匹配、进程/HTTP 执行、输出 | 生命周期拦截 |
| `src/services/lsp/` | LSP client/server manager | 代码智能插件 |
| `src/tasks/` | shell、agent、teammate、workflow、remote task | 长任务生命周期 |
| `src/utils/swarm/` | team、mailbox、计划审批、协作 todo | 多 Agent 团队 |
| `src/remote/`、`src/bridge/` | WebSocket、remote control 和外部消息接入 | 远程会话 |
| `src/ssh/` | SSH 启动、转发和远程环境 | `openclaude ssh` |
| `src/memdir/` | session/team memory 路径和索引 | 长期记忆 |
| `src/integrations/` | provider/model 描述配置与生成元数据 | 增加集成 |
| `src/ink/` | 自维护终端 renderer | 底层渲染/输入问题 |
| `src/grpc/` | 开发用 gRPC adapter | 非生产 RPC 实验 |
| `scripts/` | build、生成、诊断、发布守卫 | 工程/构建 |

`daemon`、`proactive`、`environment-runner`、`self-hosted-runner` 等目录使用独立功能开关。当前可用性需要对照 `scripts/build.ts` 的功能开关和替代模块配置。

### 17.2.1 其余顶层目录与当前状态

以下目录位于主循环外围，阅读仓库时会遇到：

| 目录 | 职责 | 当前公开构建版本的状态 |
|---|---|---|
| `src/assistant/` | KAIROS 持久 assistant 的 session 选择与历史 | `KAIROS=false`，entitlement 检查恒不通过 |
| `src/buddy/` | 终端像素伙伴、动作效果和 shot clock 观察 | `BUDDY=true`。只改变 UI 和反馈，不改变 query 执行流程 |
| `src/coordinator/` | coordinator prompt、worker tools 和 session mode | 代码已构建。还需 `CLAUDE_CODE_COORDINATOR_MODE` 运行时开启 |
| `src/daemon/` | daemon supervisor/worker registry | `DAEMON=false`，入口返回不可用 |
| `src/environment-runner/` | BYOC headless runner | 当前使用不执行功能的替代模块 |
| `src/i18n/` | locale 检测和命令文案字典 | 当前提供英文、越南文与 fallback |
| `src/jobs/` | template/job-directory turn classifier | 当前使用不执行功能的替代模块 |
| `src/migrations/` | 启动时迁移旧设置、模型名和开关 | 启动时执行。范围限于配置迁移 |
| `src/native-ts/` | 原生能力的 TypeScript 替代，如 fuzzy file index、Yoga/color helpers | 公开构建版本使用这些实现减少 native addon 依赖 |
| `src/outputStyles/` | 从 Markdown/frontmatter 加载输出风格 prompt | 与 plugin output style cache 合并 |
| `src/plugins/` | 内置插件注册表和 bundled plugin 初始化 | 与 `src/utils/plugins/` 的 marketplace/安装实现分工 |
| `src/proactive/` | proactive active/pause/tick 状态 | `PROACTIVE=false`，主入口分支被裁剪 |
| `src/proto/` | `openclaude.proto` 双向 Chat stream 契约 | 供开发 gRPC adapter 使用。npm package exports 未公开 |
| `src/self-hosted-runner/` | self-hosted worker register/poll | 当前使用不执行功能的替代模块 |
| `src/skills/` | bundled skill registry、目录加载和 MCP skill builder | 多个 bundled skills 可用。每个 skill 可以使用独立功能开关 |
| `src/upstreamproxy/` | CCR remote container 的本地 relay、CA 和子进程代理注入 | 仅 `CLAUDE_CODE_REMOTE` 等受控环境条件满足时启用 |
| `src/vim/` | Vim motion、operator、text object 和模式状态转换 | TUI 输入模式子系统 |
| `src/voice/` | voice feature/auth 可见性判断 | `VOICE_MODE=false`，当前产物不可用 |

`src/types/`、`src/schemas/` 和 `src/constants/` 保存多个模块共用的定义。`src/native-ts/` 提供替代实现。`src/__tests__/` 放置跨模块回归测试。`src/test/` 放置共享测试工具。

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
| 失败循环 | `src/query/toolFailureLoopGuard.ts` | retry 和状态变化标记 |
| 会话空闲误终止 | `src/utils/QueryGuard.ts` | `src/screens/REPL.tsx`、queryActivity |
| 权限等待计时 | `src/hooks/toolPermission/handlers/interactiveHandler.ts` | QueryGuard beginUserInteraction |
| 工具活动登记 | `src/services/tools/toolExecution.ts` | QueryGuard registerActivity |

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
| 错误重试 | `src/services/api/errors.ts`、retry helper | query 状态变化 |
| provider fallback | provider profile/fallback helpers | query recovery |

新增 provider 时应先阅读 `docs/integrations/overview.md` 和对应 `docs/integrations/how-to/`。实现前需要列出名称、默认地址、模型能力等配置，以及请求、响应和认证格式的差异。

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
| MCP 静默工具活性 | `src/services/mcp/client.ts` | MCP activity tests |
| TaskOutput 等待活性 | `src/tools/TaskOutputTool/` | TaskOutput activity tests |
| 子 Agent 进度传播 | `src/tools/AgentTool/` | 父工具 progress callback |

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
| Footer 保持挂载 | `src/components/PromptInput/KeepMounted.tsx` | PromptInputFooter |
| Footer 展示顺序 | `src/components/PromptInput/footerVisibility.ts` | PromptInputFooterLeftSide |
| StatusLine 隐藏副作用 | `src/components/StatusLine.tsx` | StatusLine active tests |

### 配置与会话

| 定位主题 | 第一站 | 后续 |
|---|---|---|
| 配置优先级 | `src/utils/settings/settings.ts` | constants/types/schema |
| 环境变量应用时机 | `src/utils/managedEnv.ts` | trust/startup |
| JSONL 写入位置 | `src/utils/log.ts` | sessionStorage |
| resume 当前消息链 | resume/log selector | transcript loaders |
| rewind 作用 | `src/commands/rewind/` | file history snapshot |
| branch 作用 | `src/commands/branch/` | parent UUID/session IDs |
| compact 记录保存 | compact service/log events | resume 时重建有效消息链 |

### 扩展系统

| 定位主题 | 第一站 | 后续 |
|---|---|---|
| MCP server 连接失败 | `src/services/mcp/` | config、auth 和连接方式 |
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
| bridge 外部消息接入 | `src/bridge/` | session 接入认证 |
| SSH | `src/ssh/` | remoteIO/unix socket auth |
| gRPC | `src/grpc/server.ts` | `scripts/start-grpc.ts` |
| 共享标题生成 | `src/utils/sessionTitle.ts` | REPL、useRemoteSession、print control request |

### 简历技术专题

| 技术主题 | 主要源码 | 主要测试 | 对应提交 |
|---|---|---|---|
| 人工等待挂起与恢复 | `src/utils/QueryGuard.ts`、interactiveHandler | QueryGuard、interactiveHandler tests | `2047fb25`，PR #1879 |
| 长运行工具活性上报 | MCP client、TaskOutputTool、AgentTool | MCP、TaskOutput、Agent routing tests | `c23b6e1c`，PR #2022 |
| Footer 零高度保活 | KeepMounted、PromptInputFooter | KeepMounted、PromptInputFooter tests | `4faf666c`，PR #1943 |
| Footer 隐藏副作用暂停 | footerVisibility、StatusLine、FooterLeftSide | StatusLine active、Footer visibility tests | `f961ae74`，PR #1963 |
| 标题 API 错误拦截 | `src/utils/sessionTitle.ts` | `src/utils/sessionTitle.test.ts` | `86fb6db8`，PR #1992 |

这些提交记录包含改动背景、评审修正和测试范围。`git show <commit>` 可以查看对应实现快照。

## 17.4 关键数据类型

### Message

内部对话事件使用 `Message` 表示。常见字段包括 role/type、content blocks、UUID、parent UUID、session metadata 和可选 usage/error。Assistant content 可以同时包含 text、thinking 和 tool use。

### Content block

一条消息内部的结构单元。常见类型包括 text、thinking、tool_use、tool_result、image/document。assistant message 可以包含多个结构块。

### Tool

可由模型调用的操作定义。每个 Tool 包含 name、description/prompt、input schema、permission check、call、并发/只读属性和 UI/result 转换函数。

### Tool use / Tool result

模型通过 Tool use 请求执行工具。本地运行结果通过 Tool result 返回。两者使用相同 ID 一一配对。配对错误会导致 provider 拒绝消息历史。

### Query

一次从现有消息开始的异步执行。一次 query 可以包含多个 model step、tool step 和 HTTP 请求。

### Turn / Step

Turn 表示用户发起的一轮交互。step 通常表示相邻两次模型调用之间的一次推进。限制代码分别统计 turn、step 和 token。

### AppState

TUI 和业务代码可订阅的当前状态快照。内容包括任务、权限请求、MCP 状态和界面数据。transcript 和模块级资源保存在其他存储中。

### Bootstrap state

初始化早期建立、供非 React 模块读取的进程/session 数据，如原始 cwd、session ID、非交互标志和 session trust。

### QueryGuard

记录主 query 的 idle、dispatching、running 和结束状态。多个 UI 操作同时提交输入时，QueryGuard 只允许其中一个启动主 query。QueryGuard 还维护最近活动时间、请求总时长、工具期限、人工交互挂起计数和 generation。

### Query activity

工具执行和模型事件通过 Query activity 接口登记活动。接口支持普通活动登记、工具期限和人工交互挂起。长运行工具的周期性进度最终通过该接口刷新主请求活动时间。

### Footer active contract

Footer 组件使用 mounted、visible 和 active 三类状态。KeepMounted 保留 React 子树。零高度容器控制终端输出。active 属性控制外部命令、刷新任务和定时器。

### API error message

模型调用可以返回带 API 错误标记的 Assistant 消息。该结果具有正常消息结构，并表示供应商请求失败。消费方需要在普通文本解析前检查错误标记。

### Command queue

保存用户 follow-up、steer、task notification 和 remote message。当前模型或工具步骤结束后，主循环按优先级读取队列。队列内容不直接写入 transcript。

### Provider descriptor

保存 provider/model 的名称、默认地址、能力和展示信息。该配置不保存某次会话的凭据或网络 client。

### Provider profile

用户可选择并持久化的一组 provider、endpoint、model 和兼容设置。程序解析 profile 后，根据其中的地址和认证信息创建 API client。

### Transport adapter

将内部 messages、tools 和 options 转换为供应商 API 请求。它还将 stream event、error 和 usage 转换为 query loop 使用的内部事件。

### Permission mode

缺少具体规则时使用的交互或自动审批策略。该设置只影响应用内授权。OS 权限和 sandbox 由独立机制控制。

### Permission rule

带来源、行为和 tool/specifier 的持久或会话授权条目。deny 通常高于 allow。

### Workspace trust

用户确认程序可以加载当前目录中的项目级 Hook、MCP、环境变量和命令。具体工具调用还要通过 permission 检查。该确认不验证项目代码的安全性。

### Sandbox

可选的 OS 进程文件/网络隔离层，主要约束 Bash 等子进程。启用状态、可用状态和强制状态是三个独立属性。

### Hook

在 Session/Prompt/Tool/Stop 等生命周期点运行的 command、HTTP 或 prompt callback，可阻止、修改、反馈或附加上下文。

### MCP

Model Context Protocol。外部 server 通过 stdio、SSE 或 HTTP 连接提供 tools、resources 和 prompts。OpenClaude 作为 client 时，将远端 tool 包装成本地 Tool 接口。

### Plugin

扩展的安装和分发单元，可以贡献 commands、agents、skills、hooks、MCP/LSP 配置。插件复用现有 Agent runtime。

### Skill

按需发现和加载的领域指令、资源和允许工具声明。Skill 文本会加入模型上下文。Skill 中提到的操作需要通过已注册 Tool 执行，并接受对应权限检查。

### Task

长生命周期执行单元的统一记录，包含 ID、type、status、progress/output 和 stop 行为。Agent、shell、workflow、remote 可有不同实现。

### Subagent

从父会话 fork 局部上下文运行的 Agent。可同步或后台，不必是独立进程。

### Teammate

拥有 team identity、mailbox、idle/active 状态和协作任务的 Agent。teammate 完成一次 prompt 后会等待新消息，并保持团队身份。

### Worktree

Git 层面的独立工作目录，用于并行修改隔离。与会话消息 branch 无关。

### Conversation branch

从某条历史消息建立新的 parent chain 和末端消息。该操作共享当前文件系统，不创建 Git branch。

### Compact

将较早的消息整理为结构化摘要。后续模型请求发送摘要和保留的近期消息。transcript 会记录压缩位置，以便 resume 时重建消息链。

### Context collapse

分阶段缩短选定的上下文片段。该流程使用独立实现，并由功能开关控制。

### Reactive compact

用于在错误后尝试压缩上下文的预留功能。当前源码快照使用禁用的替代模块，实际运行不会进入该流程。

## 17.5 容易混淆的概念对照

| 概念 A | 概念 B | 说明 |
|---|---|---|
| model | provider | model 标识模型能力。provider 提供 API、认证和协议实现。 |
| profile | API client | profile 是持久化选择配置。API client 是程序运行时创建的网络对象。 |
| query | API request | query 可以包含多次模型请求和工具调用。API request 是一次网络请求。 |
| message | stream delta | message 是完整事件。stream delta 是尚未合并的响应片段。 |
| permission | sandbox | permission 决定应用批准或拒绝操作。sandbox 使用 OS 机制限制子进程。 |
| trust | permission | trust 允许加载项目配置。permission 决定具体工具调用的执行权限。 |
| Hook | tool | Hook 在生命周期事件发生时运行。tool 由模型主动调用。 |
| Skill | plugin | Skill 提供指令和资源。plugin 是安装和分发多种扩展的包。 |
| MCP client | MCP server entry | MCP client 连接外部 server。MCP server entry 将本地工具提供给外部调用方。 |
| task | tool use | task 记录长时间运行的工作。tool use 是一次模型工具调用。 |
| subagent | teammate | subagent 使用父会话提供的局部上下文。teammate 还具有团队身份和 mailbox。 |
| background | detached process | background 表示父 query 不等待任务完成。该任务可以与父 query 运行在同一进程中。 |
| resume | branch | resume 继续现有末端消息。branch 从指定历史消息创建新链。 |
| rewind conversation | rewind files | rewind conversation 修改当前消息链。rewind files 恢复文件快照。 |
| conversation branch | Git branch/worktree | conversation branch 只创建新的消息链。Git branch/worktree 隔离文件版本。 |
| compact | truncate | compact 生成旧消息摘要并记录压缩位置。truncate 直接删除内容。 |
| retry | fallback | retry 向同一路由重发请求。fallback 改用其他 provider、model 或 profile。 |
| context overflow | output truncation | context overflow 表示输入超过窗口。output truncation 表示回答达到输出上限。 |
| 构建功能开关 | runtime setting | 构建功能开关决定代码进入构建产物的条件。runtime setting 决定已有代码的启用状态。 |
| telemetry | product network | telemetry 发送产品分析数据。模型、MCP 和 WebFetch 网络请求用于执行用户功能。 |

## 17.6 推荐学习路线

### 阶段一：建立骨架

阅读 00-04 章，并亲自追踪一次无工具请求、一次 Read 请求和一次 Bash 被拒绝请求。完成标准：能画出主循环并解释 tool pair。

### 阶段二：理解模型与上下文

阅读 05-06 章，对比一个 Anthropic request/stream 和一个 OpenAI-compatible request/stream。完成标准：能够指出描述配置保存的字段，以及 adapter 转换的请求与响应字段。

### 阶段三：理解能力与安全

阅读 07、11、14 章，手工跟踪 FileEdit、Bash 和 MCP tool 的 permission。完成标准：能说清 trust、schema、Hook、permission、safety、sandbox 的顺序。

### 阶段四：理解并发状态

阅读 08-09 章，跟踪 background agent 完成通知和运行中 follow-up。完成标准：能区分 QueryGuard、command queue、task registry 和 AppState。

### 阶段五：理解恢复与入口

阅读 10、12、13 章，检查 resume、compact、abort、headless 和 SDK。完成标准：能够说明各入口共用的 query 事件格式和独立的输入输出处理流程。

### 阶段六：形成工程判断

阅读 15、16、18 章，选择一个真实改动，写出风险、必须保持的规则、focused tests 和完整 PR 验证。完成标准包括功能、取舍、失败模式和限制的完整说明。

## 17.7 实践练习

1. 给一次 query stream 画出所有 yield 类型，并标注持久化类型。
2. 构造两个并发只读 tool use 和一个 FileEdit，预测调度顺序。
3. 找出 project settings 试图关闭 sandbox 时的判定路径。
4. 构造 symlink 指向工作区外，跟踪 FileRead 与 FileEdit 的差异。
5. 对比 OpenAI-compatible 与 Anthropic 的 tool result 请求形态。
6. 从 JSONL 某个末端事件沿 parent UUID 手工还原历史。
7. 跟踪 background Agent 从 tool return 到 task notification。
8. 在 SDK 中实现最小 `canUseTool`，确保只 resolve 一次并处理 abort。
9. 模拟 context overflow，列出 API retry 条件和 query 重启条件。
10. 修改一个 feature flag，并预测源码、bundle 和 smoke test 的变化。
11. 使用虚拟时间推演人工交互挂起、嵌套恢复和工具活性上报。
12. 推演 Footer 从建议列表切换到 Ctrl+C 反馈，再回到 StatusLine 的组件状态。
13. 构造带 API 错误标记的标题响应，并跟踪 REPL、Remote 和 SDK 的结果。

## 17.8 完整性检查矩阵

读完全部章节后，应满足以下标准：

| 领域 | 验收标准 | 对应章节 |
|---|---|---|
| 架构 | 五层职责以及配置决策与运行数据的流向 | 00 |
| 构建 | 功能开关、替代模块和 CLI/SDK bundle 形成过程 | 01、15 |
| 启动 | TUI/headless/SDK/MCP 分流 | 02、13 |
| 状态 | 四类状态的存储位置、更新时机和消息格式 | 03、09 |
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
| 简历技术专题 | 存活判定、Footer 生命周期、标题响应边界 | 18 |

只能说出功能名称时，需要继续阅读对应源码，并记录输入、状态变化、输出和错误处理流程。

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

搜索结果用于建立可能的调用关系。确认代码实际可执行时，还需要检查功能开关、dynamic import、入口参数和构建时替代模块。

## 17.10 指南索引

- [00 总体架构](00-architecture-overview.md)
- [01 运行环境与系统装配](01-repository-build-runtime.md)
- [02 启动流程](02-entrypoints-startup.md)
- [03 状态与数据模型](03-state-and-data-model.md)
- [04 会话执行流程](04-query-agent-loop.md)
- [05 上下文、Prompt、记忆与压缩](05-context-prompt-memory.md)
- [06 模型供应商接入](06-provider-model-transport.md)
- [07 工具、权限、沙箱与文件安全](07-tools-permissions-security.md)
- [08 Agent、任务、团队与编排](08-agents-tasks-orchestration.md)
- [09 终端交互](09-tui-interaction-flow.md)
- [10 配置与会话持久化](10-configuration-session-persistence.md)
- [11 MCP、插件、Hooks 与 LSP](11-mcp-plugins-hooks-lsp-commands.md)
- [12 错误与恢复](12-errors-retries-recovery-edge-cases.md)
- [13 使用入口与部署形态](13-entrypoint-modes-sdk-remote.md)
- [14 安全检查与威胁模型](14-security-model.md)
- [15 工程质量与测试方法](15-engineering-and-testing.md)
- [16 架构场景串联](16-interview-playbook.md)
- [17 源码索引与术语](17-source-map-glossary.md)
- [18 简历技术专题](18-resume-technical-deep-dives.md)
