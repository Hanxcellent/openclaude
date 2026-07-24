# 16. 简历与面试讲解稿

本章用于把源码理解转化为准确的项目表达。先遵守一个边界：可以用“项目采用”“源码实现”描述你确认过的架构；只有确实由你设计、编码、测试或评审的部分，才用“我设计/我实现”。面试官通常会沿个人贡献继续追问提交、取舍和故障，虚构 ownership 很容易被识别。

## 16.1 一句话定义

OpenClaude 是一个 TypeScript 实现的终端 Code Agent：它把多供应商 LLM 的流式输出统一成内部消息协议，在受权限、Hook 和可选 OS sandbox 约束的循环中执行文件、Shell、MCP 与子 Agent 工具，并用 React/Ink TUI、JSONL event log 和 SDK/远程入口暴露同一套核心能力。

这句话包含五个可继续展开的维度：协议适配、Agent loop、工具安全、状态与持久化、多入口复用。

## 16.2 30 秒版本

> OpenClaude 是一个多模型 Code Agent CLI。核心不是简单调用一次模型，而是维护一个可中断的流式循环：构造上下文，请求模型，解析 tool use，经过 schema、Hook、权限和沙箱执行工具，把严格配对的 tool result 回填，再继续请求直到停止。项目还处理多 provider 协议差异、上下文压缩、子 Agent/后台任务、MCP/插件扩展、React/Ink 状态同步和 JSONL 会话恢复。安全上它把模型和项目内容都视为不可信输入，采用 workspace trust、来源感知设置、最小权限和可选 OS sandbox 分层约束。

## 16.3 3 分钟架构版本

可以按五层说明：

1. **入口层**：`bin/openclaude` 最终由 Node 加载 bundle；Commander 根据参数进入 TUI、print/headless、MCP server、remote/SSH 等路径，SDK 是独立无 Ink bundle。
2. **交互与状态层**：React/Ink 管理终端交互，`AppStateStore` 保存可订阅业务状态；`QueryGuard` 保证主查询所有权；bootstrap state 保存启动后稳定的 session/cwd 等数据。
3. **Agent 执行层**：`query()` 是异步生成器。它发送模型请求并 yield 流事件；若 assistant 返回 tool use，`runTools()` 并发或串行执行，生成 tool result 后再次进入 query loop。Abort、retry、compact 和 fallback 都是显式状态迁移。
4. **能力层**：内置工具、MCP 工具、skills、plugins、Hooks 和 LSP 最终汇入统一工具/命令边界。子 Agent 复用 query loop，但拥有独立上下文、模型路由、permission context 和 task 生命周期。
5. **基础设施层**：provider adapter 将 Anthropic、OpenAI-compatible、Gemini/Vertex、Bedrock 等请求和 stream 归一化；JSONL 以 UUID/parent UUID 保存事件链；安全层处理 trust、规则、路径、sandbox、SSRF、凭据和脱敏。

最后补一个设计结论：项目没有把所有状态放进 Redux，也没有为每个入口重写 Agent loop；它通过消息协议、异步 iterator、store 和 adapter 在共享核心与入口差异之间划边界。

## 16.4 10 分钟源码走读

### 第一步：启动

从 `bin/openclaude`、`src/entrypoints/cli.tsx`、`src/entrypoints/init.ts`、`src/main.tsx` 开始：

- launcher 确认 Node runtime 并加载 `dist/cli.mjs`。
- CLI 解析参数，标记交互/非交互状态。
- `init()` 加载配置，只应用 trust 前安全环境，初始化清理、网络和预取。
- `main.tsx` 决定执行 TUI、print 或子命令。
- 交互路径先确认目录 trust，再应用完整项目环境和启动扩展。

### 第二步：一次输入

从 `src/components/PromptInput/PromptInput.tsx`、`src/utils/processUserInput/`、REPL 路径进入：

- Enter 产生 submit；若 query 正在运行，则按队列策略变成 steer/follow-up，而不是并发创建第二个主循环。
- 输入解析区分普通 prompt、slash command、`!` shell、附件和本地 command。
- UI 取得 QueryGuard 所有权，建立 AbortController 并调用 query generator。

### 第三步：模型循环

沿 `src/query.ts`、`src/QueryEngine.ts`、`src/services/api/claude.ts`：

- system prompt、tools、messages、model 参数被组装。
- provider profile 决定 transport；adapter 归一化请求。
- stream delta 转成内部 assistant message/progress。
- stop reason 是 tool use 时进入工具阶段；普通 end turn 则执行 Stop hooks 并结束。
- context overflow、max tokens、rate limit 分别触发 compact、续写或 provider fallback，且都有次数上限。

### 第四步：工具

沿 `src/tools.ts`、`src/Tool.ts`、`src/hooks/useCanUseTool.tsx` 和 `src/utils/permissions/`：

- 工具池由内置工具、动态 MCP、SDK 和 feature gate 合成。
- input schema 先阻止格式错误。
- PreToolUse Hook、permission rule、路径/命令 safety 和用户审批决定是否执行。
- 只读安全工具可并发；有副作用或不声明并发安全的工具按调度约束运行。
- 成功、错误、拒绝和 abort 都转成匹配 tool use ID 的 result。

### 第五步：状态与落盘

沿 `src/state/`、`src/utils/log.ts`、`src/utils/sessionStorage.ts`：

- stream progress 先更新 UI，不等同于完整持久消息。
- 完成消息带 UUID、parent UUID、session ID 和时间信息写入 JSONL。
- resume 读取事件，按 parent chain 重建某个叶子分支，而不是简单取文件最后 N 行。
- 大工具结果可外置，transcript 留稳定引用，以降低内存和 JSONL 体积。

这个走读应始终强调“不变量”，不要只背文件名。

## 16.5 一轮执行的白板图

```text
User input
  -> input parser / slash command dispatch
  -> QueryGuard + AbortController
  -> context + system prompt + visible tools
  -> provider adapter -> streaming API
  -> assistant blocks
       ├─ text/end_turn -> Stop hooks -> persist -> idle
       ├─ tool_use -> validate -> hook -> permission -> sandbox -> tool.call
       │              -> tool_result -> persist -> next model step
       └─ error -> retry / compact / lower cap / fallback / terminal error
  -> AppState/UI/SDK consumer receives the same normalized messages
```

面试官若打断，可从任意箭头展开相应章节。

## 16.6 最值得讲的设计点

### 1. 统一消息语义，而不是强行统一供应商 API

不同 provider 对 system、tool calling、thinking、usage、stream event 和 token limit 的支持不同。项目在 adapter 层接受差异，再归一化为内部 `Message`/content block。这样 query loop 不需要为每个 provider 重写，但 adapter 仍可做能力降级。

权衡：统一层减少上层复杂度，但兼容 provider 的非标准行为会集中到较大的 transport 模块，需要强分类测试。

### 2. Async generator 连接 stream 和状态消费者

query 不是返回最终字符串，而是逐步 yield assistant delta、tool progress、retry 信息和最终消息。TUI、headless 和 SDK 可以按自身方式消费同一执行流，AbortSignal 也能向下传播。

权衡：流式用户体验和复用性较好，但 iterator 关闭、partial state、异常恢复和 tool pair repair 更复杂。

### 3. 权限是来源感知的多阶段决策

workspace trust 防项目配置提前执行；permission rule 控制具体能力；文件/Bash safety 做确定性检查；OS sandbox 限制实际进程。deny、policy 和危险路径优先级高于普通自动批准。

权衡：安全性高于单一确认框，但规则来源、模式和 UI 解释成本增加。

### 4. 子 Agent 复用核心循环但隔离局部状态

subagent 不是另一个 CLI 进程的必然别名。它可在进程内复用 query，拥有独立 prompt、tools、model、abort 和 task record；foreground/background 转换通过 task signal 协调。worktree 只在需要 Git 文件隔离时使用。

权衡：复用传输和状态基础设施、开销低，但必须防止全局 env/config/cache 交叉污染。

### 5. JSONL 是 append event log

消息不是覆盖写成一个大 JSON 数组。UUID/parent UUID 支持 resume、分支、compact boundary 和中断恢复；追加写减少每轮重写成本。

权衡：写入简单且可恢复，但读取要做 projection、父链重建、损坏行容错和大结果外置。

## 16.7 高频架构追问

### Q1：为什么不用一个 while 循环加几次 HTTP 请求？

核心确实是循环，但产品级实现还需处理 stream partial event、工具并发、权限异步等待、中止、重试、压缩、后台通知和多消费者。`async generator + explicit transitions` 使这些中间状态可观察和可测试。

### Q2：Agent loop 何时终止？

模型返回无 tool use 的正常 stop、达到 turn/step/token 限制、用户 abort、不可恢复 API error、工具失败保护触发或 Hook 明确终止。Stop hooks 可反馈阻止一次普通结束，但失败路径受循环保护。

### Q3：如何保证 tool result 不乱序？

每个 call/result 通过 tool use ID 配对。可并发执行不代表随意拼接协议；调度层收集结果并按 provider 可接受的消息结构回填。中断时也生成拒绝/中止 result 修复未闭合 pair。

### Q4：为什么工具需要同时有 schema 和 permission？

Schema 回答“参数结构是否合法”，permission 回答“该合法操作是否被允许”。合法的 `FileRead({path:'/etc/passwd'})` 仍可能越权，两者不能合并。

### Q5：怎么处理模型反复调用失败工具？

失败签名按工具和输入归一化跟踪。相同或相似失败连续出现时注入警告或终止，避免 token 和外部副作用无限消耗。普通 retry 不应覆盖 deterministic tool failure。

### Q6：为什么 auto compact 不直接删除旧消息？

需要保留任务目标、关键决策、文件状态和未闭合 tool pair。compact 生成结构化摘要，并记录 boundary，使 prompt projection 变小但 transcript 仍可审计/恢复。

### Q7：如何区分 context overflow 和 max output tokens？

前者是输入加预期输出超出 context，需要压缩/减少历史；后者是请求已被接受但回答被截断，可降低 cap、提高默认 cap 或注入 continuation。混用会导致无效重试。

### Q8：多 provider 的最大难点是什么？

不是 base URL，而是消息角色、system 表达、tool schema/call/result、thinking block、stream framing、usage 和错误语义不一致。项目用 descriptor/profile 做配置，用 transport adapter 做协议转换，用统一错误分类驱动上层恢复。

### Q9：为什么 AppState 之外还有 bootstrap state 和 QueryGuard？

生命周期不同。AppState 是可订阅 UI/业务状态；bootstrap state 是进程启动后稳定且非 React 的 session/cwd 数据；QueryGuard 是并发所有权协议。塞进一个 store 会模糊写入时机和责任。

### Q10：运行中再次输入会怎样？

不是创建两个主 query。输入进入统一 command queue，根据类型作为 steer、follow-up 或本地控制命令处理；队列与 QueryGuard 共同保持单主循环语义。

### Q11：ESC 为什么复杂？

它按优先级取消 modal、输入子模式、permission、前台 query 或任务。只有最内层未消费时才向外传播，避免为了关闭搜索框而误杀模型请求。

### Q12：后台 Agent 如何回到主会话？

启动时注册 task 并立即向父工具返回 task ID。后台 iterator 独立消费；完成后写 output、更新 task status，并将结构化 task notification 放入主 command queue，下一安全点并入上下文。

### Q13：worktree 和 conversation branch 有何区别？

worktree 是 Git 文件系统隔离，给并行 Agent 独立工作目录；conversation branch 是 JSONL 消息父链的新叶子，不创建 Git branch，也不隔离文件修改。

### Q14：MCP 如何接入？

配置解析并建立 stdio/SSE/HTTP transport，完成 initialize 和 capability discovery，将远端 tool 包装为本地 Tool 接口。连接状态、OAuth、resource 和 elicitation 由 MCP service 管理；执行仍进入统一 schema/permission/result 流程。

### Q15：Plugin、Skill、Command、MCP 的区别？

Plugin 是能力包和分发单元，可贡献 commands、agents、skills、hooks、MCP/LSP 配置；Skill 是按需加载的指令/资源；Command 是用户输入触发的操作描述；MCP 是外部进程/服务协议。它们可以组合，但不是同一抽象。

### Q16：sandbox 是否默认保证所有操作隔离？

不是。它是可选 OS 约束，主要包装 shell 进程；未启用时依赖 permission 和应用层路径检查。即使已请求，依赖不可用默认也可能警告后降级，只有 `failIfUnavailable` 才硬失败。

### Q17：如何防恶意仓库？

交互模式先做 workspace trust，项目级危险 env、Hooks、MCP 和 command 不应在确认前执行。确认后仍有具体 permission 和 sandbox。headless/bypass 将信任责任交给调用者，适合受控 CI，不适合直接执行未知仓库。

### Q18：SSRF 如何处理？

HTTP Hook 的自定义 DNS lookup 阻止 private/link-local/metadata 范围并检查 IPv4-mapped IPv6；loopback 为本地策略服务而有意允许。使用全局代理时需要依赖代理自己的 DNS/egress policy。

### Q19：会话恢复如何避免读到错误分支？

事件有 UUID 和 parent UUID。加载器选择目标 leaf 后沿 parent 链 projection，而不是假设 JSONL 最后一行就是唯一历史；branch 和 rewind 都产生明确的新链语义。

### Q20：如何测试这类系统？

以状态迁移和协议不变量为中心：构造 provider stream，断言消息序列、tool pair、重试上限、abort、task status 和持久化。纯函数测边界，TUI 测事件，真实 provider 做少量契约验证，构建再检查 Node 产物和 stub/externals。

## 16.8 深入追问与限制

### reactive compact 现在可用吗？

当前源码快照中 `src/services/compact/reactiveCompact.ts` 是 feature-gated stub，`isReactiveCompactEnabled()` 返回 false。可讲标准 auto compact/context collapse，不能声称 open build 已实际启用 reactive compact。

### gRPC 是生产 server 吗？

不是。它由开发脚本启动，使用 insecure transport、内存 Map session 和有限治理；生产化至少需要 TLS、认证授权、限流、持久 session、审计和多租户隔离。

### MCP server 入口等同完整 Agent API 吗？

不是。当前入口主要重暴露选定工具，构造较薄的非交互上下文直接调用；它不是把完整 TUI query loop 作为 MCP 服务发布。

### 无 telemetry 是否等于无网络？

不是。open build 的 analytics/sink 是 no-op，并有 privacy build check；模型 API、OAuth、MCP、WebFetch、插件和更新仍是显式功能网络流量。

### 权限能完全阻止 prompt injection 吗？

不能。权限降低可执行影响，不能可靠判断自然语言意图。用户若批准任意 Bash，进程可在当前 OS 权限内组合读文件和出网；应叠加 sandbox、最小规则和部署隔离。

## 16.9 个人贡献表达模板

只选择能用 commit、测试或设计记录证明的模板：

### 若贡献 provider 兼容

> 我负责的是 `[具体 provider/adapter]`。问题是 `[具体协议差异]`，我没有改上层 query loop，而是在 transport 边界把 `[字段/stream/error]` 归一化，并补了 `[测试文件/真实模型路径]`。这样避免影响其他 provider。主要权衡是 `[兼容性与严格校验]`。

### 若贡献工具/权限

> 我修改了 `[具体工具或 permission stage]`。我先保留 deny/policy/safety 的优先级，只在 `[精确条件]` 放行；测试覆盖正常路径以及 symlink、abort、拒绝和 headless/SDK 场景。没有把一次批准扩大成持久全局规则。

### 若贡献 UI

> 我负责 `[具体交互]`，事实状态位于 `[store/queue/guard]`，组件只订阅所需 selector。重点处理 stream 高频更新、ESC 优先级、窄终端和恢复状态，并用 `[测试和手工终端]` 验证。

### 若主要是二次开发或学习项目

> 我基于 OpenClaude 做了 `[明确改动/集成/部署]`，并系统梳理了它的 Agent loop、provider adapter 和安全模型。上游核心架构不是我原创；我的工作集中在 `[可证明范围]`。

这种表述比笼统说“我开发了 OpenClaude”更可信，也给面试官明确追问边界。

## 16.10 演示脚本

准备一个 5 分钟可重复 demo：

1. 在临时 Git 仓库启动交互 CLI，解释 trust。
2. 要求 Agent 阅读一个小模块并修改一处有测试的 bug。
3. 展示 Read/Grep 后的 FileEdit permission 或 diff。
4. 运行 focused test，指出 Bash permission/sandbox 状态。
5. 用 `/context` 或相关 UI 展示 token/context。
6. 结束后 resume 会话，解释 JSONL parent chain。
7. 可选切换 provider profile，展示上层 loop 无变化。

提前准备失败路径：API key 无效、模型不支持 tool calling、sandbox 依赖缺失、终端过窄。演示中出现故障时按错误层分类，比重启程序更能证明理解。

## 16.11 面试前自检

- 能否不看代码画出输入到 tool result 的完整时序？
- 能否说清 AppState、bootstrap、QueryGuard、command queue 的生命周期差异？
- 能否解释三种“分支”：query retry transition、conversation branch、Git worktree？
- 能否解释 provider profile、model descriptor、transport 的区别？
- 能否列出 trust、permission、safety、sandbox 四层边界？
- 能否解释同步 Agent、后台 Agent、teammate 和 shell task？
- 能否指出当前 open build 的 stub/禁用能力？
- 能否用一个真实 commit 说明自己的问题、方案、测试和权衡？
- 能否诚实回答尚未读过或未运行过的部分？

完成这些问题后，再按 [17 源码导航与术语表](17-source-map-glossary.md) 做定向复习。
