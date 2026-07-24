# 01. 仓库、构建与运行时

## 1. 技术栈

- 语言：TypeScript strict、ESM。
- 交互 UI：React + 自维护 Ink 派生实现（`src/ink/`）。
- 开发/构建/测试：Bun。
- 安装后运行时：Node.js `>=22.0.0`。
- CLI 参数：Commander。
- schema：Zod。MCP JSON Schema 额外用 Ajv。
- 子进程：Node APIs、`execa` 等。
- API：Anthropic SDK + OpenAI/Gemini/云平台适配。

`package.json` 的 `bin.openclaude` 指向 `bin/openclaude`。发布文件主要包含 launcher、bundled `dist/cli.mjs`、`dist/sdk.mjs` 和声明文件。用户执行的是构建产物。

## 2. 目录职责

| 目录 | 作用 |
|---|---|
| `src/entrypoints/` | CLI、SDK、MCP、生成类型等入口 |
| `src/bootstrap/` | 轻量早期状态和启动隔离 |
| `src/screens/`, `src/components/` | TUI 页面和组件 |
| `src/ink/` | 终端布局、diff、输入、滚动、选择、自定义 renderer |
| `src/query.ts`, `src/query/` | 主循环及其状态/保护器 |
| `src/services/api/` | 模型 client、provider transport、retry/error |
| `src/integrations/` | vendor/gateway/model descriptor registry |
| `src/tools/` | 内置工具，每个工具通常有 Tool、prompt、UI、tests |
| `src/services/tools/` | 通用工具执行编排 |
| `src/utils/permissions/` | 权限规则、路径/命令安全、模式 |
| `src/tasks/` | shell、agent、remote、workflow 等任务类型 |
| `src/services/mcp/` | MCP transport、连接、资源/命令/工具发现 |
| `src/utils/plugins/` | 插件 marketplace、缓存、loader、组件桥接 |
| `src/services/lsp/` | LSP manager/client/diagnostics |
| `src/utils/settings/` | settings schema、来源、合并、缓存、policy |
| `src/utils/sessionStorage.ts` | transcript 的写、读、索引、恢复 |
| `src/commands/` | slash command 与对话内本地命令 |
| `src/server/`, `src/grpc/`, `src/ssh/`, `src/remote/` | 非默认部署形态 |
| `scripts/` | build、生成 descriptor 产物、验证脚本 |
| `docs/integrations/` | provider 扩展规范 |
| `web/` | Astro 文档网站，独立 package |

## 3. 构建系统

`scripts/build.ts` 用 Bun bundler 生成入口 bundle。关键特征：

1. 把 React、reconciler、scheduler 锁定到一致的 production 模块，避免同一 bundle 出现两份 React。
2. 通过 `bun:bundle` 的 `feature()` 做编译期死代码消除。
3. 分别构建 CLI 和 SDK 入口。
4. 对需要保留为外部依赖的模块做显式校验。
5. 生成宏（如版本）供运行时代码读取。

### 3.1 feature gate 属于构建控制

常见代码：

```ts
if (feature('SOME_FEATURE')) {
  const module = require('./feature.js')
}
```

构建脚本会为不同产物给 gate 常量值，关闭分支能被 DCE。因此：

- 关闭功能的依赖可以不进入 open bundle。
- 顶层 import 可能破坏 DCE，所以很多模块用受 gate 保护的 lazy `require`。
- 测试可通过 Bun feature 变体覆盖不同构建形态。
- 发布 bundle 的模块集合由构建期开关决定。

### 3.2 两层功能控制

- **构建层**：`feature('X')` 决定代码的存在状态。
- **运行/实验层**：环境变量、settings、`getFeatureValue_CACHED_MAY_BE_STALE()` 决定已有代码的启用状态。

典型功能同时需要两层为真。

## 4. Node 发布运行时与 Bun 构建运行时

- Bun 提供快速 bundle、test 和 compile-time feature。
- Node 22 是用户环境的稳定运行契约。
- 代码不能无意依赖 Bun-only API。`bun:bundle` 是构建宏的特殊例外。
- `bin/openclaude` 负责运行时版本、安装形态和 bundle 定位。用户使用 Node 启动发布产物。

## 5. 自维护终端渲染层

`src/ink/` 提供完整的终端渲染实现。它包含：

- React reconciler 与自定义终端 DOM。
- Yoga layout。
- front/back `Frame` 和屏幕 diff。
- ANSI tokenizer、grapheme width、双向文本。
- scroll box、selection、hit test、cursor。
- terminal raw input、resize、focus、alternate screen。

`src/ink/renderer.ts` 复用 `Output` 和字符 cache。当窗口 resize、remount、首帧或前帧被 selection 污染时，才选择高写入比/full redraw 路径。该优化直接影响长对话滚动和 streaming 的 CPU/闪烁表现。

## 6. 生成代码和源数据

provider 集成采用 descriptor + generator：

1. descriptor 定义 vendor/gateway/brand/model。
2. `bun run integrations:generate` 生成 inventory/manifest。
3. `src/integrations/index.ts` 懒加载 generated artifacts。
4. registry 查询自动触发 `ensureIntegrationsLoaded()`。

provider 变更需要修改 descriptor 并重新生成产物。生成流程会覆盖对 generated 文件的手工修改。

## 7. 测试布局

测试通常与源码共置为 `*.test.ts(x)`，另有 `src/__tests__/` 和根 `tests/`。常见层级：

- 纯函数单测：settings merge、provider error parsing、path safety。
- 状态机测试：QueryGuard、query transitions、retry breaker。
- Tool contract 测试：input validation、permission、output mapping。
- React/Ink 组件测试：PromptInput、REPL lifecycle。
- provider 测试：协议形状和模型推荐。
- smoke：打包产物能在 Node 环境启动。

## 8. 常用验证命令

```bash
bun run build
bun run smoke
bun run check
bun run typecheck
bun run typecheck:type-tests
bun test ./src/path/file.test.ts
bun run test:provider
bun run test:provider-recommendation
```

触及 `web/` 时另跑 `bun run web:typecheck` 和 `bun run web:build`。

## 9. 阅读建议

- 阅读工作应从导出符号和 `buildTool()` 定义开始。UI 文案仅用于呈现行为。
- 看见 lazy `require` 时先查 gate。
- 看见 `_DEPRECATED` 时需要检查调用者。该标记可能对应公开兼容层。
- 遇到 provider 分支，先判断是 descriptor 元数据、env compatibility，还是不可消除的协议差异。

下一章：[02 入口与启动链路](02-entrypoints-startup.md)。
