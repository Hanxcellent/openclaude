# 16. 简历与面试讲解稿

本章用于把源码理解转化为准确的项目表达。“项目采用”和“源码实现”适用于已经确认的架构。“我设计”和“我实现”适用于本人参与设计、编码、测试或评审的部分。面试官通常会沿个人贡献继续追问提交、取舍和故障。虚构 ownership 容易被识别。

## 16.1 一句话定义

OpenClaude 是一个 TypeScript 实现的终端 Code Agent：它把多供应商 LLM 的流式输出统一成内部消息协议，在受权限、Hook 和可选 OS sandbox 约束的循环中执行文件、Shell、MCP 与子 Agent 工具，并用 React/Ink TUI、JSONL event log 和 SDK/远程入口暴露同一套核心能力。

这句话包含五个可继续展开的维度：协议适配、Agent loop、工具安全、状态与持久化、多入口复用。

## 16.2 30 秒版本

> OpenClaude 是一个多模型 Code Agent CLI。核心运行时维护可中断的流式循环。循环依次构造上下文、请求模型、解析 tool use、执行安全检查、运行工具和回填 tool result。项目还处理多 provider 协议差异、上下文压缩、子 Agent/后台任务、MCP/插件扩展、React/Ink 状态同步和 JSONL 会话恢复。安全层把模型和项目内容视为不可信输入，并采用 workspace trust、来源感知设置、最小权限和可选 OS sandbox 进行分层约束。

## 16.3 3 分钟架构版本

可以按五层说明：

1. **入口层**：`bin/openclaude` 最终由 Node 加载 bundle。Commander 根据参数进入 TUI、print/headless、MCP server、remote/SSH 等路径，SDK 是独立无 Ink bundle。
2. **交互与状态层**：React/Ink 管理终端交互，`AppStateStore` 保存可订阅业务状态。`QueryGuard` 保证主查询所有权。bootstrap state 保存启动后稳定的 session/cwd 等数据。
3. **Agent 执行层**：`query()` 是异步生成器。它发送模型请求并 yield 流事件。若 assistant 返回 tool use，`runTools()` 并发或串行执行，生成 tool result 后再次进入 query loop。Abort、retry、compact 和 fallback 都是显式状态迁移。
4. **能力层**：内置工具、MCP 工具、skills、plugins、Hooks 和 LSP 最终汇入统一工具/命令边界。子 Agent 复用 query loop。每个子 Agent 拥有独立上下文、模型路由、permission context 和 task 生命周期。
5. **基础设施层**：provider adapter 将 Anthropic、OpenAI-compatible、Gemini/Vertex、Bedrock 等请求和 stream 归一化。JSONL 以 UUID/parent UUID 保存事件链。安全层处理 trust、规则、路径、sandbox、SSRF、凭据和脱敏。

最后补充设计结论。项目通过消息协议、异步 iterator、store 和 adapter 划分共享核心与入口差异。不同生命周期的状态由对应存储管理。各入口复用统一 Agent loop。

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

- Enter 产生 submit。query 运行期间的输入按队列策略转换为 steer 或 follow-up。主会话始终保持单个执行循环。
- 输入解析区分普通 prompt、slash command、`!` shell、附件和本地 command。
- UI 取得 QueryGuard 所有权，建立 AbortController 并调用 query generator。

### 第三步：模型循环

沿 `src/query.ts`、`src/QueryEngine.ts`、`src/services/api/claude.ts`：

- system prompt、tools、messages、model 参数被组装。
- provider profile 决定 transport。adapter 归一化请求。
- stream delta 转成内部 assistant message/progress。
- stop reason 是 tool use 时进入工具阶段。普通 end turn 则执行 Stop hooks 并结束。
- context overflow、max tokens、rate limit 分别触发 compact、续写或 provider fallback，且都有次数上限。

### 第四步：工具

沿 `src/tools.ts`、`src/Tool.ts`、`src/hooks/useCanUseTool.tsx` 和 `src/utils/permissions/`：

- 工具池由内置工具、动态 MCP、SDK 和 feature gate 合成。
- input schema 先阻止格式错误。
- PreToolUse Hook、permission rule、路径/命令 safety 和用户审批共同决定执行权限。
- 只读安全工具可并发。有副作用或不声明并发安全的工具按调度约束运行。
- 成功、错误、拒绝和 abort 都转成匹配 tool use ID 的 result。

### 第五步：状态与落盘

沿 `src/state/`、`src/utils/log.ts`、`src/utils/sessionStorage.ts`：

- stream progress 先更新 UI，不等同于完整持久消息。
- 完成消息带 UUID、parent UUID、session ID 和时间信息写入 JSONL。
- resume 读取事件，并按 parent chain 重建指定叶子分支。
- 大工具结果可外置，transcript 留稳定引用，以降低内存和 JSONL 体积。

这个走读应始终强调不变量、状态迁移和文件职责。

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

### 1. 统一消息语义并保留供应商差异

不同 provider 对 system、tool calling、thinking、usage、stream event 和 token limit 的支持不同。项目在 adapter 层处理差异，再归一化为内部 `Message`/content block。query loop 由此可以复用。adapter 根据供应商能力执行降级。

成本：兼容 provider 的非标准行为会集中到较大的 transport 模块。该模块需要细分的分类测试。

### 2. Async generator 连接 stream 和状态消费者

query 通过异步生成器逐步 yield assistant delta、tool progress、retry 信息和最终消息。TUI、headless 和 SDK 可以按自身方式消费同一执行流。AbortSignal 可以沿调用链向下传播。

成本：iterator 关闭、partial state、异常恢复和 tool pair repair 需要额外状态管理。

### 3. 权限是来源感知的多阶段决策

workspace trust 防项目配置提前执行。permission rule 控制具体能力。文件/Bash safety 做确定性检查。OS sandbox 限制实际进程。deny、policy 和危险路径优先级高于普通自动批准。

成本：规则来源、模式和 UI 需要提供完整解释信息。

### 4. 子 Agent 复用核心循环并隔离局部状态

subagent 可以在进程内复用 query。它拥有独立 prompt、tools、model、abort 和 task record。foreground/background 转换通过 task signal 协调。worktree 用于需要 Git 文件隔离的场景。

约束：实现必须防止全局 env、config 和 cache 交叉污染。

### 5. JSONL 是 append event log

消息以 JSONL 事件增量追加。UUID/parent UUID 支持 resume、分支、compact boundary 和中断恢复。追加写减少每轮重写成本。

成本：读取过程需要完成 projection、父链重建、损坏行容错和大结果外置。

## 16.7 高频架构追问

### Q1：AsyncGenerator 与显式状态迁移的作用

核心控制结构是循环。产品级实现还需处理 stream partial event、工具并发、权限异步等待、中止、重试、压缩、后台通知和多消费者。`async generator + explicit transitions` 使这些中间状态可观察和可测试。

### Q2：Agent loop 的终止条件

终止条件包括无 tool use 的正常 stop、turn/step/token 限制、用户 abort、不可恢复 API error、工具失败保护和 Hook 明确终止。Stop hooks 可以反馈并阻止一次普通结束。失败路径受循环保护。

### Q3：tool result 的顺序保证

每个 call/result 通过 tool use ID 配对。调度层收集并发结果，并按 provider 可接受的消息结构回填。中断时也会生成拒绝或中止 result，以修复未闭合 pair。

### Q4：schema 与 permission 的独立职责

Schema 校验参数结构。permission 校验操作授权。`FileRead({path:'/etc/passwd'})` 可以通过 schema 校验，并因权限范围被拒绝。两项检查分别承担独立职责。

### Q5：重复失败工具调用的处理

失败签名按工具和输入归一化跟踪。相同或相似失败连续出现时注入警告或终止，避免 token 和外部副作用无限消耗。普通 retry 不应覆盖 deterministic tool failure。

### Q6：auto compact 的信息保留规则

compact 需要保留任务目标、关键决策、文件状态和未闭合 tool pair。该过程生成结构化摘要并记录 boundary。prompt projection 随之缩小。transcript 保持可审计和可恢复。

### Q7：context overflow 与 max output tokens 的分类

context overflow 表示输入加预期输出超出 context。该错误需要压缩或减少历史。max output tokens 表示请求已被接受，并在回答达到上限后截断。该错误可以通过降低 cap、提高默认 cap 或注入 continuation 处理。两类错误混用会导致无效重试。

### Q8：多 provider 的主要难点

主要难点来自消息角色、system 表达、tool schema/call/result、thinking block、stream framing、usage 和错误语义的差异。项目用 descriptor/profile 管理配置，用 transport adapter 完成协议转换，并用统一错误分类驱动上层恢复。

### Q9：AppState、bootstrap state 与 QueryGuard 的分工

生命周期不同。AppState 是可订阅 UI/业务状态。bootstrap state 是进程启动后稳定且非 React 的 session/cwd 数据。QueryGuard 是并发所有权协议。塞进一个 store 会模糊写入时机和责任。

### Q10：运行中输入的处理

运行中输入进入统一 command queue。系统根据类型将其作为 steer、follow-up 或本地控制命令处理。队列与 QueryGuard 共同保持单主循环语义。

### Q11：ESC 的取消优先级

它按优先级取消 modal、输入子模式、permission、前台 query 或任务。最内层未消费事件时，事件才向外传播。该顺序防止关闭搜索框的操作误杀模型请求。

### Q12：后台 Agent 的主会话通知

启动时注册 task 并立即向父工具返回 task ID。后台 iterator 独立消费。完成后写 output、更新 task status，并将结构化 task notification 放入主 command queue，下一安全点并入上下文。

### Q13：worktree 与 conversation branch 的作用范围

worktree 是 Git 文件系统隔离，给并行 Agent 独立工作目录。conversation branch 是 JSONL 消息父链的新叶子，不创建 Git branch，也不隔离文件修改。

### Q14：MCP 接入流程

配置解析并建立 stdio/SSE/HTTP transport，完成 initialize 和 capability discovery，将远端 tool 包装为本地 Tool 接口。连接状态、OAuth、resource 和 elicitation 由 MCP service 管理。执行进入统一 schema/permission/result 流程。

### Q15：Plugin、Skill、Command 与 MCP 的职责

Plugin 是能力包和分发单元，可以贡献 commands、agents、skills、hooks、MCP/LSP 配置。Skill 是按需加载的指令和资源。Command 是用户输入触发的操作描述。MCP 是外部进程或服务协议。四者使用独立抽象，并可组合使用。

### Q16：sandbox 的默认覆盖范围

Sandbox 是可选 OS 约束，主要包装 shell 进程。未启用时由 permission 和应用层路径检查提供限制。依赖不可用时默认可能警告后降级。`failIfUnavailable` 会把该情况转换为硬失败。

### Q17：恶意仓库防护

交互模式先执行 workspace trust。项目级危险 env、Hooks、MCP 和 command 在确认后执行。具体 permission 和 sandbox 在确认后生效。headless/bypass 将信任责任交给调用者，适用于受控 CI 环境和已验证仓库。

### Q18：SSRF 防护

HTTP Hook 的自定义 DNS lookup 阻止 private/link-local/metadata 范围，并检查 IPv4-mapped IPv6。loopback 被明确允许，以支持本地策略服务。使用全局代理时需要依赖代理自己的 DNS/egress policy。

### Q19：会话恢复的分支选择

事件有 UUID 和 parent UUID。加载器选择目标 leaf 后沿 parent 链执行 projection。branch 和 rewind 都产生明确的新链语义。

### Q20：系统测试方法

以状态迁移和协议不变量为中心：构造 provider stream，断言消息序列、tool pair、重试上限、abort、task status 和持久化。纯函数测边界，TUI 测事件，真实 provider 做少量契约验证，构建再检查 Node 产物和 stub/externals。

## 16.8 深入追问与限制

### reactive compact 的当前状态

当前源码快照中 `src/services/compact/reactiveCompact.ts` 是 feature-gated stub，`isReactiveCompactEnabled()` 返回 false。可讲标准 auto compact/context collapse，不能声称 open build 已实际启用 reactive compact。

### gRPC 的成熟度

gRPC 当前是开发原型。它由开发脚本启动，并使用 insecure transport、内存 Map session 和有限治理。生产化至少需要 TLS、认证授权、限流、持久 session、审计和多租户隔离。

### MCP server 入口的能力范围

当前 MCP server 入口主要重暴露选定工具。该入口构造较薄的非交互上下文并直接调用工具。完整 TUI query loop 不在其服务范围内。

### telemetry 与产品网络流量

open build 的 analytics/sink 是 no-op，并包含 privacy build check。模型 API、OAuth、MCP、WebFetch、插件和更新属于显式功能网络流量。

### 权限对 prompt injection 的限制范围

不能。权限降低可执行影响，不能可靠判断自然语言意图。用户若批准任意 Bash，进程可在当前 OS 权限内组合读文件和出网。应叠加 sandbox、最小规则和部署隔离。

## 16.9 个人贡献表达模板

只选择能用 commit、测试或设计记录证明的模板：

### 若贡献 provider 兼容

> 我负责 `[具体 provider/adapter]`。问题是 `[具体协议差异]`。上层 query loop 保持原有实现。我在 transport 边界完成 `[字段/stream/error]` 归一化，并补充 `[测试文件/真实模型路径]`。该范围可以避免影响其他 provider。主要权衡是 `[兼容性与严格校验]`。

### 若贡献工具/权限

> 我修改了 `[具体工具或 permission stage]`。我先保留 deny/policy/safety 的优先级，只在 `[精确条件]` 放行。测试覆盖正常路径以及 symlink、abort、拒绝和 headless/SDK 场景。没有把一次批准扩大成持久全局规则。

### 若贡献 UI

> 我负责 `[具体交互]`，事实状态位于 `[store/queue/guard]`，组件只订阅所需 selector。重点处理 stream 高频更新、ESC 优先级、窄终端和恢复状态，并用 `[测试和手工终端]` 验证。

### 若主要是二次开发或学习项目

> 我基于 OpenClaude 完成了 `[明确改动/集成/部署]`，并系统梳理了它的 Agent loop、provider adapter 和安全模型。上游核心架构由项目维护者设计。我的工作范围是 `[可证明范围]`。

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

- 可以独立画出输入到 tool result 的完整时序。
- 可以说明 AppState、bootstrap、QueryGuard、command queue 的生命周期差异。
- 可以解释 query retry transition、conversation branch 和 Git worktree 三类分支。
- 可以解释 provider profile、model descriptor 和 transport 的区别。
- 可以列出 trust、permission、safety 和 sandbox 四层边界。
- 可以解释同步 Agent、后台 Agent、teammate 和 shell task。
- 可以指出当前 open build 的 stub 和禁用能力。
- 可以用一个真实 commit 说明问题、方案、测试和权衡。
- 可以准确说明尚未读过或未运行过的部分。

完成这些问题后，再按 [17 源码导航与术语表](17-source-map-glossary.md) 做定向复习。
