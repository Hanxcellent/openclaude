# 15. 工程质量、构建与测试方法

本项目的主要工程难点是跨入口和 provider 保持 Agent 语义一致。相关入口包括交互 CLI、headless、SDK、MCP 和远程会话。验证策略同时覆盖类型、构建产物、状态机、协议兼容和安全不变量。

## 15.1 开发与发布运行时

- 源码、依赖管理、构建脚本和测试使用 Bun。
- 发布后的 `bin/openclaude` 启动 Node.js，要求 Node `>=22.0.0`。
- TypeScript 目标为 ES2023、ESM、strict mode，module resolution 使用 bundler。
- CLI 和 SDK 分别构建为 `dist/cli.mjs`、`dist/sdk.mjs`。
- React/Ink 属于 CLI TUI。SDK bundle 明确 externalize UI 依赖。

这种双运行时结构要求同时验证“Bun 能构建/测试”和“Node 能实际加载发布产物”。仅运行 `tsc` 不能发现 bundle resolver、stub、externals 或 Node launcher 问题。

## 15.2 构建流水线

`scripts/build.ts` 的核心阶段是：

```text
读取 package version
  -> 扫描并转换 feature('FLAG')
  -> 注入 MACRO 构建常量
  -> 替换 no-telemetry/internal/optional modules
  -> 绑定 production React runtime
  -> Bun.build CLI
  -> 校验 build result 与 stub marker
  -> Bun.build SDK（UI external）
  -> 校验 SDK 未泄漏 React/Ink 等依赖
```

### 构建期 feature

源码使用 `feature()`。当前 Bun 会在 JS plugin 之前处理 `bun:` namespace。构建脚本因此在 `onLoad` 阶段直接删除 import，并把调用替换为布尔字面量。dead-code elimination 随后移除未启用分支。

由此得到两个阅读规则：

1. 发布产物的功能集合由构建结果决定。
2. 修改 feature 名或调用形态时，必须同时考虑预处理正则和 source guard tests。

### Stub 与 open build

部分上游内部模块、原生 addon 或未镜像模块由 build plugin 提供 stub。构建会记录 stub marker。测试会检查允许范围，并识别被静默替换的缺失 import。

这是一种发行兼容策略。stub 不提供完整功能。调用 stub 的 live path 应明确返回“unavailable”错误。空成功结果会掩盖缺失能力。

### CLI 保留标识符的原因

CLI 只做 whitespace/syntax minify，不做 identifier mangling，因为错误处理和工具执行存在 `constructor.name` 等兼容判断。SDK 保持未压缩，以便构建后做 import leak 检查。这个细节说明 minification 配置也是运行语义的一部分。

## 15.3 类型边界

普通类型检查：

```bash
bun run typecheck
```

它覆盖 `src/**/*`。`noImplicitAny` 当前为 `false`。类型安全评估需要检查参数的实际推导结果。

公开 SDK 类型、条件类型和 compile-time contract 另由 `tsconfig.type-tests.json` 与 `scripts/typecheck-type-tests.ts` 校验：

```bash
bun run typecheck:type-tests
```

修改 `src/entrypoints/sdk.d.ts`、generated core types、AppState type contract 或 optional module 类型时，两类检查都需要运行。

## 15.4 测试形态

当前仓库的测试主要与源码共置，文件命名为 `*.test.ts`/`*.test.tsx`，使用 `bun:test`。测试数量较大，覆盖以下层级：

| 层级 | 典型对象 | 主要断言 |
|---|---|---|
| 纯函数 | path、redaction、token、provider mapping | 输入输出、边界值、跨平台 |
| 状态机 | QueryEngine、task、permission、retry | 状态迁移、次数上限、中止 |
| React/Ink | dialog、picker、status、input | 渲染文本、按键、callback |
| 协议 | provider、MCP、SDK、bridge | request shape、stream event、tool pair |
| 文件/进程 | settings、JSONL、plugin、Bash | 临时目录、锁、退出码、清理 |
| 构建守卫 | scripts 下 tests | stub、feature、externals、隐私 |
| 回归/安全 | `src/__tests__` 与专项 tests | 已知漏洞和历史 bug 不变量 |

### full test 的并发限制

`bun run test:full` 使用 `--max-concurrency=1`。大量测试会修改 `process.env`、cwd、全局 config cache、bootstrap state、mock transport 和临时 credential store。串行模式降低跨文件全局状态干扰，并与 CI 保持一致。

focused test 可以并发提供快速反馈。涉及全局状态的改动还应运行对应 suite 的串行组合。

## 15.5 核心命令的真实覆盖

| 命令 | 实际行为 | 适用时机 |
|---|---|---|
| `bun run build` | 构建 CLI + SDK，并执行 bundle guards | 入口、feature、import、发布面变化 |
| `bun run smoke` | build 后用 Node 执行 `--version` | 所有运行时/构建变化 |
| `bun run typecheck` | TypeScript no-emit 检查 | 所有 TS 变化 |
| `bun run typecheck:type-tests` | 独立公开类型契约检查 | SDK/公共类型变化 |
| `bun test path` | 单个或少量 focused tests | 开发内循环 |
| `bun run test:full` | 单并发全量测试 | 合并前行为验证 |
| `bun run deadcode` | knip 文件/依赖检查 | 模块、依赖、入口变化 |
| `bun run check` | smoke + deadcode + full tests | 通用合并门禁 |
| `bun run test:provider` | API provider 与 context 专项 | provider/transport 变化 |
| `bun run test:provider-recommendation` | profile/recommendation 专项 | provider 选择变化 |
| `bun run security:pr-scan` | 检查 PR 意图和敏感改动模式 | 权限、安全、发布前 |
| `bun run verify:privacy` | 检查 build 中不存在非预期 phone-home | analytics/network/build 变化 |
| `bun run doctor:runtime` | 本机 Node/Bun/依赖/环境诊断 | 安装和运行时问题 |

`bun run check` 不包含 `typecheck`、provider 专项、type tests、privacy 或 web build，所以不能把它称为“所有 CI 检查”。应按改动范围组合命令。

## 15.6 CI 门禁

`.github/workflows/pr-checks.yml` 将检查分成多个 job，主要包括：

- smoke 和完整单元测试。
- provider 与 provider recommendation tests。
- TypeScript typecheck 与 type tests。
- PR intent/security scan。

release workflow 还会重新执行测试、smoke、打包和发布，并确认 npm `latest` 指向预期版本。发布后的 dist-tag 验证用于检测 registry 状态尚未收敛的情况。

Web 是独立子项目，修改 `web/` 要额外运行：

```bash
bun run web:typecheck
bun run web:build
```

## 15.7 按改动选择测试

### 修改 Query/Tool 循环

最小集合：

```bash
bun test src/query/<相关测试>.test.ts
bun test src/tools/<Tool>/<相关测试>.test.ts
bun run typecheck
```

合并前增加 `bun run check`。必须覆盖成功、拒绝、abort、tool error、并发和严格 `tool_use/tool_result` 配对。

### 修改 provider

先确认该变化属于公共语义、Anthropic transport 还是 OpenAI-compatible adapter。运行：

```bash
bun run test:provider
bun run test:provider-recommendation
bun run typecheck
bun run smoke
```

还应对实际 provider/model 做一次真实请求，因为 mock 很难覆盖 SSE framing、兼容字段和服务端 token 限制。PR 中必须说明精确测试路径，不能用一个 provider 的成功推断全部第三方兼容。

### 修改设置、权限或安全

测试必须包含来源优先级与负例：

- deny 覆盖 allow。
- project setting 受 policy/user ownership 约束。
- symlink、`..`、大小写、Windows/UNC 变体。
- headless、SDK 和交互审批行为保持一致。
- sandbox 不可用时是 fail-open warning 还是 `failIfUnavailable` hard fail。

运行 focused tests 和 `bun run security:pr-scan`。涉及网络或 analytics 时运行 `bun run verify:privacy`。

### 修改 TUI

优先测试状态与事件，不依赖完整 ANSI snapshot：

- 输入按键对应的 action。
- dialog 的选项和取消调用正确 callback。
- terminal 宽度、窄屏、超长字符串和 CJK 宽度。
- stream 更新保持稳定 key，并避免重复消息。

手工验证应至少覆盖普通终端、重定向/非 TTY 和 resize。若改变用户可见 UI，贡献规则要求 PR 提供截图。

### 修改 SDK/入口/构建

运行：

```bash
bun run typecheck:type-tests
bun run build
bun run smoke
bun run deadcode
```

SDK 还要验证 async iterator 关闭、中止、环境恢复、permission callback 和多个 session。入口变化需要分别检查 `--print`、structured output、TUI 和 SDK。`--version` 仅覆盖 launcher 和最小 bundle 加载路径。

### 只修改文档

文档验证需要检查相对链接、命令和源码路径。feature 状态需要对照 `scripts/build.ts`。纯 Markdown 变更可以省略全量 TypeScript 测试。验证记录需要明确说明省略的代码测试及原因。

## 15.8 测试 Agent 状态机的方法

状态机测试应围绕可观察 transition：

```text
给定：消息历史、provider stream、permission mode、AbortSignal
当：收到 assistant delta / tool_use / error / abort
则：产出消息序列、任务状态、持久化记录和最终停止原因
```

关键不变量包括：

1. 每个 `tool_use` 在成功、拒绝和中止路径中都恰有一个匹配 ID 的 `tool_result`。
2. Abort 不应被 retry 捕获并重新请求。
3. context/output recovery 均有次数上限。
4. 后台任务完成通知最多入队一次。
5. 失败的 Stop hook 不应让 API error 无限循环。
6. resume/branch 不应破坏 parent UUID 链。
7. stream delta 不应重复形成持久化 assistant message。

测试 fixture 应显式构造 provider event。自然语言结果无法覆盖 partial JSON、stop reason 和 usage 更新。

## 15.9 Mock、临时状态和清理

本项目大量模块使用 memoize 或模块级 singleton。测试修改以下状态后必须恢复：

- `process.env`、cwd、argv 和 HOME 派生配置目录。
- bootstrap session state、AppState store 和 query singleton。
- settings cache、provider client cache、OAuth/keychain prefetch。
- fake timers、AbortController 和长期 background iterator。
- MCP/LSP server、WebSocket、child process 和文件 watcher。

优先使用 `mkdtemp` 或测试 helper 创建精确临时目录，并在 `finally`/`afterEach` 清理。涉及共享全局 mutation 的 suite 已有串行锁模式。新测试应沿用相邻测试的 helper 和 reset 机制。

## 15.10 调试路线

### 启动失败

1. `bun run doctor:runtime` 检查 Node/Bun/外部命令。
2. `bun run build` 区分源码构建与 launcher 问题。
3. `node dist/cli.mjs --version` 绕开 `bin/openclaude`。
4. 查看 invalid config 的具体 source/path。

### Provider 请求失败

1. 确认 active provider profile、model、base URL 和 key 来源。
2. 同时检查统一错误分类和 provider 原文。
3. 对照 request adapter 对 system、tools 和 thinking 字段的剥离与转换行为。
4. 区分 API retry 与 query transition。确认当前 compact/fallback 状态。
5. 使用 `bun run dev:profile` 或明确 `--provider` 复现。

### 工具卡住

1. 判断等待的是 permission、Hook、tool process 还是 task notification。
2. 检查 AbortSignal 沿 query → runTools → tool.call 的传播链。
3. 检查进程 stdout/stderr drain 与 timeout。
4. MCP 工具检查 server connection state。background agent 检查 task registry。
5. 确认 tool_result 已生成。缺失结果可能使下一轮 provider 拒绝消息历史。

### UI 与状态不一致

1. 先判断事实状态位于 bootstrap、AppState、QueryGuard 还是模块 queue。
2. 检查 selector 的订阅字段和原地修改产生的通知缺失。
3. 检查消息 UUID/key 的稳定性。
4. 区分 transient progress 与持久 message。
5. 对 resume 场景检查 JSONL 与内存 projection 的一致性。

## 15.11 性能观察点

性能优化集中在下列路径：

- startup profiler checkpoint：配置、网络、auth、Git/IDE 预取。
- API preconnect：在 UI/命令准备期间重叠 TCP/TLS。
- prompt cache：system/tool/message 前缀稳定性。
- microcompact、context collapse 和外置大型 tool result。
- React selector 与 virtual message list，避免 stream 每个 delta 全树重渲染。
- 并发只读 tools 与后台 agent，不阻塞前台输入。
- SDK bundle dependency graph，避免将整个 TUI 带入宿主。

优化前先定义指标：首屏时间、首 token、turn 总耗时、cache read/write、峰值 RSS、render FPS、工具排队时间。只看单个函数 microbenchmark 容易忽略 provider 和终端 I/O 的主导成本。

## 15.12 贡献流程

根据 `CONTRIBUTING.md`：

1. 非平凡 feature、refactor、依赖或 runtime 变化先建立 issue 共识。
2. 一个 PR 只解决一个问题，避免顺带格式化和重命名。
3. 行为变化增加测试。用户可见设置、命令和 provider 行为更新文档。
4. PR 描述写明影响、关联 issue、精确命令、provider 路径和 UI 截图。
5. 处理 CodeRabbit 和 maintainer 反馈后再请求复审。

评审重点包括有限失败模式、跨入口语义一致性、第三方 provider 兼容性、全局状态恢复，以及构建期不可用功能的 live path 隔离。

## 15.13 源码定位

- `package.json`：脚本、发布 exports、Node/Bun 约束。
- `scripts/build.ts`：CLI/SDK bundle、feature、stub 和 macro。
- `scripts/no-telemetry-plugin.ts`、`scripts/verify-no-phone-home.ts`：隐私构建守卫。
- `scripts/typecheck-type-tests.ts`、`tsconfig.type-tests.json`：公共类型测试。
- `knip.json`：dead-code 入口和有意保留项。
- `.github/workflows/pr-checks.yml`：PR CI。
- `.github/workflows/release.yml`：发布与 registry 验证。
- `src/test/`、`src/__tests__/`：共享 fixture 和跨模块回归。
- `CONTRIBUTING.md`、`AGENTS.md`：贡献约束和验证要求。
