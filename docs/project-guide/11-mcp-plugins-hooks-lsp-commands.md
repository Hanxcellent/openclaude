# 11. MCP、插件、Hooks、LSP 与命令系统

OpenClaude 有多条扩展路径。它们最终可能都给模型增加能力，但加载时机、信任边界和控制流不同。

| 机制 | 本质 | 主要产物 | 是否独立进程/网络 |
|---|---|---|---|
| Slash command / skill | 文本展开或本地控制命令 | prompt、状态变更、Ink UI | 不一定 |
| Plugin | 本地扩展包和分发单位 | commands、skills、agents、hooks、MCP、LSP、output styles | 取决于组件 |
| MCP | 标准协议客户端 | 动态 tools、prompts、resources | 通常是 |
| Hook | 生命周期拦截器 | 放行、阻断、改写、附加上下文 | command/http/prompt/agent/callback |
| LSP | 代码语义服务 | definition、references、hover 等 | 子进程或 socket |

## 11.1 Command 的统一类型

所有可解析命令实现 `CommandBase`，再落入三种执行类型。

### `prompt`

把 Markdown、MCP prompt 或动态 skill 展开成内容块，送入模型。它可声明：

- 参数、允许工具、模型和 effort。
- 来源、插件元数据、适用文件 glob。
- 是否允许用户调用、是否允许模型调用。
- `context: inline | fork`；fork 时通过指定 agent 在独立上下文执行。
- skill 激活期间注册的 hooks。

SkillTool 只暴露符合条件且可由模型调用的 prompt command。是否能在 `/` 自动补全中显示、用户能否手动调用、模型能否调用，是三个独立条件。

### `local`

惰性加载 TypeScript `call()`，返回文本、compact 结果或 `skip`。它可以支持非交互模式，但不能渲染 Ink 组件。

### `local-jsx`

惰性加载并返回 React/Ink 节点，用于 `/config`、`/resume`、`/model` 等交互对话框。无 TTY 或远程桥接场景不能假设它可执行。

### 查找和安全过滤

命令可按 `name`、用户显示名或 alias 查找。`isEnabled` 控制当前开关，`availability` 控制认证/提供商适用范围，`isHidden` 只影响发现界面。

远程模式有两级过滤：

- `REMOTE_SAFE_COMMANDS` 决定远程 REPL 中保留哪些本地命令。
- Bridge 输入允许 prompt command；`local` 必须进入显式 allowlist；`local-jsx` 一律拒绝，避免移动端输入弹出本地终端 UI。

相关源码：`src/types/command.ts`、`src/commands.ts`、`src/utils/slashCommandParsing.ts`。

## 11.2 Plugin 是能力包，不是新执行引擎

一个插件可包含：

```text
plugin-root/
  .claude-plugin/plugin.json
  commands/
  skills/
  agents/
  output-styles/
  hooks/hooks.json
  .mcp.json
  .lsp.json
  settings.json
```

manifest 可以声明额外路径、内联 command/hook、MCPB 包以及 MCP/LSP 配置。`LoadedPlugin` 只是标准化后的索引；具体组件由各自 loader 再转成系统已有对象：

```text
Plugin command  -> Command
Plugin agent    -> AgentDefinition
Plugin hook     -> HookMatcher
Plugin MCP      -> ScopedMcpServerConfig
Plugin LSP      -> ScopedLspServerConfig
Output style    -> system prompt customization
```

因此，插件工具调用仍经过第 7 章的权限和 tool execution pipeline；插件 hook 仍经过本章 hook 聚合器；插件 MCP server 仍由 MCP connection manager 管理。

### 装载过程

1. 从安装记录、市场和启用设置解析候选插件。
2. 读取并 schema 校验 manifest；没有 manifest 时可构造最小 manifest。
3. 解析默认目录及 manifest 附加路径。
4. 校验解析后的组件路径仍位于插件根目录，拒绝路径逃逸。
5. 分别加载 command、skill、agent、hook、MCP、LSP、output style。
6. 将错误作为 `PluginError` 保存；一个组件失败不必使其他插件全部失败。

插件级 `settings.json` 优先于 manifest 内联 settings，但整体位于普通用户/项目/flag/policy 配置之下。

### 启用、安装与信任

“已安装”和“在某 scope 启用”分开记录。同一插件可安装在用户层，却在项目层显式启用或本地层禁用。管理策略可以：

- 限制已知 marketplace。
- 阻止 marketplace 或具体 plugin。
- 只允许插件提供某些 customization surface。
- 控制跨 marketplace 依赖；根 marketplace 的 allowlist 不向下传递信任。

安装 UI 明确提示：插件可能包含 MCP server、文件和可执行软件，使用者必须信任来源。项目启动检查要在“信任当前目录”之后运行，防止仓库内容在信任前触发安装或执行。

官方 marketplace 名称是保留名称，并校验其 GitHub 组织/URL，防止第三方仅通过命名伪装为官方源。

### 热重载

`refreshActivePlugins()` 是一次协调刷新：

1. 清理 plugin、command、skill、agent 等缓存。
2. 完整加载插件，随后加载依赖其缓存的 commands 和 agents。
3. 预热 MCP/LSP 配置。
4. 原子更新 AppState 插件和 agent 定义。
5. 增加 `mcp.pluginReconnectKey`，触发 MCP 重建。
6. 重建 LSP manager 配置。
7. 最后原子替换 plugin hooks。

被删除插件的 hooks 会立即裁剪；新增 hooks 等 `/reload-plugins` 完整装载。旧 hook 集保持有效直到新集合就绪，避免刷新窗口中 Stop hook 突然消失。

相关源码：`src/utils/plugins/pluginLoader.ts`、`src/utils/plugins/refresh.ts`、`src/services/plugins/pluginOperations.ts`。

## 11.3 MCP 配置与连接状态机

MCP server 配置支持以下 transport：

| Transport | 实现/用途 |
|---|---|
| `stdio` | 启动本地子进程，通过 stdin/stdout JSON-RPC |
| `sse` | 旧版 HTTP SSE transport |
| `http` | Streamable HTTP |
| `ws` | WebSocket，可带代理和 mTLS 选项 |
| `sdk` | SDK 控制 transport 或进程内 transport |
| `sse-ide` | IDE 内部连接 |

配置 scope 包含 local、user、project、dynamic、enterprise、managed、plugin/Claude AI 等来源。合并后还要经过管理员规则：例如 `allowManagedMcpServersOnly` 会排除非托管 server，disabled 配置会产生 `disabled` 状态而不是尝试连接。

每个 server 的状态为：

```text
pending -> connected
       -> needs-auth
       -> failed
disabled（不进入连接）
```

SSE 等可恢复连接失败后最多自动重试 5 次，指数退避从 1 秒增长并封顶 30 秒。连接更新先放入批次，约 100 ms 合并写入一次 AppState，避免多个 server 同时回调造成高频渲染。

### 能力发现

连接成功后并行发现：

- `tools/list` -> `MCPTool`。
- `prompts/list` -> prompt command/skill。
- `resources/list` -> `ListMcpResourcesTool` 与 `ReadMcpResourceTool` 可访问的数据。
- server capabilities、instructions 和 server info。

工具名标准化为：

```text
mcp__<normalized-server-name>__<normalized-tool-name>
```

这既避免与内建工具冲突，也让权限规则能精确匹配 server 和 tool。SDK 进程内 server 可在约定条件下跳过此前缀。

server description/instructions 最多送入约 2,048 字符，避免自动生成的 OpenAPI 文档直接占满上下文。工具结果还经过 MCP 自己的二进制持久化/截断，再进入通用的大结果预算。

### 动态能力变化

MCP `tools/list_changed`、`prompts/list_changed`、`resources/list_changed` 通知会：

1. 清除对应 server 的 memoized list。
2. 重新拉取该类能力。
3. 按 `mcp__server__` 前缀替换旧项。
4. 批量写回 AppState。

不会把旧工具与新工具简单相加，否则 server 删除工具后模型仍可能调用陈旧定义。

### 工具调用

MCP 工具仍是标准 `Tool`：先 schema 校验、权限判断、PreToolUse hook，再调用 MCP client，最后运行 PostToolUse/Failure hook并形成 tool result。默认调用超时为 300 秒，可由 `MCP_TOOL_TIMEOUT` 覆盖。

若 Streamable HTTP 返回“HTTP 404 且 JSON-RPC code -32001”，系统把它识别为 MCP session 过期，清连接缓存、重新连接并重试，而不是把所有普通 404 都当作 session 失效。

### OAuth

远端 transport 可使用 MCP OAuth：

- metadata URL 必须为 HTTPS。
- 回调校验 `state`，防止 CSRF。
- access/refresh token 和 client 信息存入 secure storage。
- 单次 OAuth 请求有独立超时。
- 401 会触发 refresh/重新认证路径，并把连接状态改为 `needs-auth`。
- `needs-auth` 有短期磁盘缓存，避免启动时对已知需登录 server 重复失败探测。

某些 server 用 HTTP 200 返回 OAuth error body，代码会先识别 error schema，再把它规范化为失败响应。

### Elicitation、roots 和 channel

MCP server 可请求：

- roots：返回当前允许的工作根。
- elicitation：要求用户接受、拒绝或取消结构化输入；hook 可在 UI 前后参与决定。
- channel notification：经 channel allowlist、策略和权限 relay 后入消息队列。

server 发来的内容不是自动可信用户输入。channel 消息要带来源包装并经过 server gate；permission relay 有明确 pending request map 和用户响应对应关系。

相关源码：`src/services/mcp/client.ts`、`types.ts`、`config.ts`、`auth.ts`、`useManageMCPConnections.ts`。

## 11.4 Hook 生命周期

支持的事件集合包括：

```text
PreToolUse, PostToolUse, PostToolUseFailure, PermissionDenied,
Notification, UserPromptSubmit, SessionStart, SessionEnd,
Stop, StopFailure, SubagentStart, SubagentStop,
PreCompact, PostCompact, PermissionRequest, Setup,
TeammateIdle, TaskCreated, TaskCompleted,
Elicitation, ElicitationResult, ConfigChange,
WorktreeCreate, WorktreeRemove, InstructionsLoaded,
CwdChanged, FileChanged
```

来源可以是 settings、plugin、skill/frontmatter、SDK callback 或会话动态注册。matcher 通常按工具名或事件相关名称筛选，多个匹配项再聚合。

### 执行类型

| 类型 | 行为 |
|---|---|
| `command` | 启动 shell 命令，stdin 传 HookInput，解析 stdout JSON |
| `http` | 受 SSRF 防护的 HTTP 调用 |
| `prompt` | 让模型按给定 prompt 做判断 |
| `agent` | 启动带工具的 verifier agent |
| `callback` | SDK/内部注册的异步函数 |

每个 hook 有独立超时，同时受调用方 AbortSignal 约束。长任务可先返回 `{async:true, asyncTimeout}`，注册到异步 hook registry；完成通知再进入消息队列。

### 结构化结果

同步输出经 Zod schema 校验。通用字段可：

- `continue: false` 阻止后续流程并给出 `stopReason`。
- 隐藏命令输出或显示 system warning。
- `approve`/`block`。

事件专用输出可：

- PreToolUse 改写 input、改变权限决策、附加上下文。
- PermissionRequest 返回 allow/deny，并附带会话权限更新。
- PostToolUse 改写 MCP tool output。
- UserPromptSubmit/SessionStart/SubagentStart 附加上下文。
- PermissionDenied 请求 retry。
- ConfigChange 阻止外部配置更新。
- WorktreeCreate 返回新的 worktree path。
- FileChanged/CwdChanged 更新监听路径。

聚合器必须区分 blocking error、non-blocking error、cancelled 和 success。非阻断 hook 失败应可观察，但不能默认中止主循环；阻断决定则必须在工具真正执行前生效。

### Stop hook 防递归

Stop hook 可以阻止结束并要求模型继续，但需要携带“当前正在处理 stop hook”的状态，防止新一轮再次无条件触发同一 Stop hook而无限循环。目标继续条件还会与 Stop 结果共同决定是否真正结束。

### 可观测性

hook started/progress/response 使用独立事件通道，不混入模型消息流。没有 SDK handler 时最多缓冲 100 条；SessionStart 和 Setup 始终发出，其他事件仅在 `includeHookEvents` 或 remote 模式启用。完整 stdout/stderr 仍写 debug log。

相关源码：`src/utils/hooks.ts`、`src/types/hooks.ts`、`src/schemas/hooks.ts`、`src/query/stopHooks.ts`。

## 11.5 LSP 集成

LSP server 只从启用插件加载，不直接接受普通用户/项目 settings 中任意 server 配置。每个配置声明命令、参数、extension-to-language 映射、transport、环境变量和启动/停止策略。

全局 manager 的状态为：

```text
not-started -> pending -> success
                     -> failed --显式重试--> pending
```

初始化只解析配置和建立 extension map；具体 language server 在某文件首次请求时惰性启动。多个 server 声明同一扩展名时，当前实现使用注册顺序中的第一个。

`LSPTool` 支持 definition、references、hover、document/workspace symbols、implementation 和 call hierarchy。它：

- 是只读、并发安全、可延迟加载的标准 Tool。
- 仅在至少一个 LSP 配置健康时启用。
- 调用前检查文件存在、普通文件、读取权限和约 10 MB 大小限制。
- 等待尚未完成的 manager 初始化。
- 用 `didOpen`/`didClose` 管理文档，并收集 diagnostics 被动反馈。
- 单个 server 初始化失败不会阻止其他 server；整体 shutdown 聚合错误。

裸模式/简化非交互入口跳过 LSP。插件刷新会递增 generation，使旧初始化 promise 不能覆盖新 manager 状态，并尽力关闭旧子进程。

相关源码：`src/services/lsp/manager.ts`、`LSPServerManager.ts`、`LSPServerInstance.ts`、`src/tools/LSPTool/`。

## 11.6 扩展能力进入主循环的总路径

```text
配置/插件/MCP
   -> 规范化为 Command / AgentDefinition / Tool / HookMatcher
   -> AppState 或 bootstrap registry
   -> system prompt 暴露可用能力
   -> 模型选择 tool/skill
   -> 权限 + hooks + execution pipeline
   -> 结果持久化并回送模型
```

关键不变量：

1. 扩展只能增加定义，不能绕过统一权限执行器。
2. 插件路径必须限制在插件根目录。
3. MCP 动态 list 变化必须替换旧 server 项。
4. Hook 输出必须经过结构校验，文本 stdout 不能直接成为权限决定。
5. LSP 对文件的访问仍受读取权限约束。
6. 无 TTY/remote 模式不能执行需要本地 Ink UI 的命令。

下一章集中整理错误分类、重试、取消、上下文溢出和资源耗尽等特殊场景。
