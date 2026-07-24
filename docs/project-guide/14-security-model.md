# 14. 安全边界与威胁模型

OpenClaude 的安全目标不是证明模型输出可信，而是把模型视为可能生成错误或恶意操作的非可信决策源，再由确定性代码限制操作范围。其防护由 workspace trust、设置来源、工具权限、路径校验、进程沙箱、网络限制和凭据隔离共同构成。任何单层都不是完整安全边界。

## 14.1 需要保护什么

主要资产包括：

- 工作区源码、工作区外文件、Git 元数据和用户配置。
- API key、OAuth token、云凭据、代理认证信息和系统 keychain。
- 主机进程、网络可达的内网服务、云 metadata endpoint。
- 会话 JSONL、工具持久化结果、调试日志和诊断报告中的隐私数据。
- 权限规则、MCP/Hook/插件配置等会改变后续执行能力的控制面数据。
- 远程 session 的 ingress token、WebSocket 身份和审批通道。

主要非可信输入包括：

1. 用户打开的仓库及其中的 `.openclaude`/兼容配置、指令和 Hooks。
2. 模型返回的文本、工具名和工具参数。
3. MCP server、插件、技能及远程消息。
4. HTTP 响应、网页内容、Git 输出和 IDE 传入内容。
5. 恢复的历史 JSONL，以及从外部复制来的 permission rule。

因此，“用户本人启动了 CLI”不等于“当前仓库中的自动执行配置可信”。

## 14.2 防护层次

```text
workspace trust
  -> 设置来源与 feature/policy 约束
  -> 可见工具集合与 permission mode
  -> tool input schema + tool-specific permission
  -> 路径/命令/域名的确定性安全检查
  -> 用户或宿主审批
  -> 可选 OS sandbox
  -> 执行后的结果清理、持久化与脱敏
```

前一层允许继续，只表示可以进入下一层，不能跳过后面的判定。例如用户信任目录后，危险 Bash 仍可需要审批；用户允许一次 Bash 后，沙箱仍可阻止它访问被禁目录。

## 14.3 Workspace trust

`src/components/TrustDialog/TrustDialog.tsx` 在交互入口询问用户是否信任当前目录。它还扫描项目级 MCP、Hooks、Bash 权限、可执行 command/skill、凭据 helper 和危险环境变量，使提示反映当前目录可能触发的执行能力。

信任状态由 `src/utils/config.ts` 管理：

- 非 home 目录写入项目配置的 `hasTrustDialogAccepted`。
- 从当前路径向父目录遍历；信任父目录隐含信任其子目录。
- home 目录接受仅记在当前进程的 bootstrap state，不持久化整棵 home 树的信任。
- `isPathTrusted(dir)` 用于检查非当前 cwd 的目标目录，不读取 session-only trust。
- 会话中 trust 只从 `false` 变为 `true`，所以正结果可 latch；负结果不缓存，以便用户刚确认后立即生效。

### 启动前后的环境变量边界

`src/entrypoints/init.ts` 在 trust 前只调用 `applySafeConfigEnvironmentVariables()`。`src/utils/managedEnv.ts` 区分可信来源与项目来源：危险的 `PATH`、动态库预加载、代理和命令相关变量不能在确认目录前由项目配置注入。

`NODE_EXTRA_CA_CERTS` 是特殊情况。TLS store 可能在第一次连接时缓存，所以 `src/utils/caCertsConfig.ts` 会提前读取全局配置和 user settings，但明确不读取 project settings，避免恶意仓库在 trust 前注入 CA。

交互模式在 TrustDialog 通过后才执行 `applyConfigEnvironmentVariables()`。print/headless 和 bypass 模式没有交互 trust 对话框，源码明确将调用者视为已建立信任，然后应用完整环境。因此自动化执行不应直接对任意下载目录运行 `-p` 或 bypass permission。

### Trust 的能力边界

Trust 保护的是“是否启用项目控制面”，不是文件真实性验证，也不是隔离机制。它不能检测仓库中隐蔽的提示注入或恶意依赖；接受后项目级 Hook、MCP、环境和命令具备进入后续执行链的资格。

## 14.4 设置来源和策略所有权

设置合并不是简单的最后写入者获胜。`src/utils/settings/` 为 user、project、local、policy、flags 等来源保留身份，安全判定可以只读取可信来源。

典型规则包括：

- sandbox 是否开启只从 user、local、flag 和 policy 等可信来源读取，项目配置不能关闭外层要求的沙箱。
- policy 可限制插件、MCP、permission bypass、additional directory 和 network domain。
- deny rule 优先于 allow rule；上层 policy deny 不能被会话 allow 覆盖。
- 插件依赖跨 marketplace 时，仅根 marketplace 的显式 allowlist 生效，不做传递信任。
- provider 由宿主管理时，项目设置中的 provider-routing 环境变量会被剥离。

保留来源身份的设计使安全策略能回答“谁授权了这项能力”，而不只是“合并后的值是什么”。

## 14.5 模型到工具的安全边界

工具执行入口在调用 `tool.call()` 前至少经过以下步骤：

1. 当前会话是否向模型暴露该工具。
2. 返回的工具名能否映射到注册工具或 MCP 工具。
3. `inputSchema`/Zod 校验是否通过。
4. PreToolUse hooks 是否阻止、改写或请求确认。
5. 工具自身 `checkPermissions()` 和全局 permission rules。
6. 交互 UI、SDK callback 或 remote permission bridge 给出决定。
7. 可选 sandbox 包装实际进程。

工具结果被视为数据返回模型，而不是本地控制指令。MCP 工具名不能进入内置工具的特殊白名单，外部 channel 消息也会被包裹为“非用户输入、按不可信内容处理”的系统说明。

## 14.6 Permission mode 与规则

权限模式决定缺少显式规则时的默认行为，但不应被理解为单个布尔量：

- `default`：按工具风险、规则和用户确认处理。
- `acceptEdits`：工作区内普通编辑可自动批准，危险路径仍不会直接放行。
- `plan`：限制为规划所需的读操作和少数状态工具。
- `dontAsk`：不能交互询问时拒绝未预先批准的操作。
- `bypassPermissions`：显著扩大能力，仍受 hard safety、policy kill switch 和可能的 sandbox 约束。
- `delegate`/`auto` 等专用模式会叠加 agent 或分类器语义，具体集合由当前 build feature 决定。

规则由 tool name 和可选 specifier 组成。匹配时 deny 优先，命令规则使用结构化解析/前缀语义，文件规则按规范化路径匹配。会话级“允许一次”和持久规则是不同更新类型，不应把一次批准自动写成长期授权。

自动审批分类器只是额外信号。确定性危险模式、policy deny、敏感文件和不允许被分类器批准的 safety check 仍具有更高优先级。

## 14.7 文件系统安全

`src/utils/permissions/filesystem.ts` 和 `pathValidation.ts` 实现文件工具与 shell 静态分析共用的路径边界。

### 判定顺序

写路径的核心顺序是：

1. 匹配 deny rule。
2. 识别受控的内部可写目录，如当前 plan、scratchpad、agent memory 和 job 目录。
3. 执行不可绕过的危险路径检查。
4. 判断是否位于 cwd/additional directory，并结合 `acceptEdits`。
5. 若在工作区外，检查 sandbox write allowlist。
6. 匹配显式 allow rule。
7. 其余情况要求批准或拒绝。

将 safety check 放在 cwd 自动允许之前很关键，否则仓库内的 `.git/config`、`.openclaude/settings.json` 可能借 `acceptEdits` 被静默修改。

### 规范化与逃逸防护

- 展开 `~`，但不支持 `~username`。
- 规范化 `.`、`..`、平台分隔符和大小写比较。
- 同时检查输入路径、已存在祖先的 realpath 和 symlink 解析结果。
- glob 先提取不含通配符的 base directory，再检查目录边界。
- Windows 路径额外处理盘符、UNC 和容易绕过只读判定的路径形式。
- 路径包含关系按 segment 判断，不能用字符串 `startsWith` 代替。

被保护的典型对象包括 shell profile、`.gitconfig`、`.gitmodules`、`.mcp.json`、OpenClaude 配置、`.git`、IDE 设置目录以及 `.openclaude`/`.claude` 控制目录。技能目录可生成只针对某一个 skill 的窄规则，且会拒绝 `..` 和 glob 元字符，防止建议规则意外扩大。

TOCTOU 仍是需要理解的残余风险：静态检查和实际打开文件之间，攻击者可能替换 symlink。OS sandbox 能进一步缩小影响，但应用层路径检查本身不是原子 capability handle。

## 14.8 Bash 与进程安全

Bash 安全不是通过字符串查找几个危险词实现。`src/utils/bash/ast.ts` 用 tree-sitter 解析命令，只有能可靠还原 `argv[]` 的语法才可进入精确规则匹配；变量展开、命令替换、复杂重定向和无法确定的组合会降低自动批准条件。

只读命令也按参数语义检查。例如同一二进制可因写入参数、配置覆盖、输出重定向或 UNC 路径而变成有副作用的命令。shell rule 匹配不能把 `git status` 的许可扩展为任意 `git`。

子进程环境由 `src/utils/subprocessEnv.ts` 集中组装，可移除宿主私有变量、注入受控代理，并保留显式 provider 选择的优先级。用户 shell 配置、可执行文件解析和命令本身仍可能执行任意代码，所以没有 sandbox 的 permission approval 本质上是用户授权该进程使用当前 OS 权限。

## 14.9 OS Sandbox

`src/utils/sandbox/sandbox-adapter.ts` 把项目设置转换为 `@anthropic-ai/sandbox-runtime` 配置，主要能力是：

- 文件写 allowlist、写 deny-within-allow 和读 deny。
- 网络 domain allowlist、Unix socket 和本地监听限制。
- 对 shell 命令提供进程级文件/网络约束。
- 把 WebFetch 规则和 managed domains 合并到网络配置。
- 无条件保护 settings、插件/命令控制目录和可能导致后续 unsandboxed Git 逃逸的配置。

需要区分三种状态：

| 状态 | 实际含义 |
|---|---|
| 未配置 sandbox | Bash 按 permission 执行，但没有 OS 隔离 |
| 已启用且依赖可用 | 命令由 sandbox runtime 包装 |
| 已启用但不可用 | 默认警告后可能无沙箱运行；`failIfUnavailable` 才是硬失败 |

`allowUnsandboxedCommands` 默认可允许用户明确请求在沙箱外运行。高安全环境应通过 policy 禁止，并设置 `failIfUnavailable`。Linux/WSL/macOS 能力不完全等价，`enabledPlatforms` 也可能使某平台不启用沙箱。

因此“项目支持 sandbox”不能表述为“所有工具永远被 sandbox”。文件工具主要依靠应用层路径权限；sandbox 重点约束 shell 子进程，而且是否强制由可信设置决定。

## 14.10 网络与 SSRF

WebFetch 对 URL scheme、domain permission、重定向和响应大小执行专用检查。Hook 的 HTTP 请求还使用 `src/utils/hooks/ssrfGuard.ts`：

- 阻止 IPv4 私网、link-local、CGNAT 和 `0.0.0.0/8`。
- 阻止 IPv6 ULA、link-local、unspecified，以及映射到受限 IPv4 的地址。
- 有意允许 loopback，支持本地策略服务。
- 通过受控 DNS lookup 检查解析结果，降低直接访问云 metadata 的风险。

该 guard 的注释明确说明：全局代理负责 DNS 时，目标主机检查可能由代理边界接管；sandbox 网络代理有自己的 domain allowlist。这意味着部署代理时必须同时审计代理的解析和出站策略。允许 loopback 也意味着本机服务不是默认隔离资产。

网页、MCP resource 和 Hook 输出都可能包含 prompt injection。域名获准只表示允许获取数据，不表示内容可信；内容仍应作为外部数据进入模型上下文。

## 14.11 插件、MCP、Hook 与 LSP

这些扩展通常在 OpenClaude 进程权限下执行，是重要供应链边界：

- 项目级扩展只在 workspace trust 后启动。
- 插件安装显示来源和 trust warning；本地路径、Git 来源和 marketplace 元数据经过规范化与 containment 检查。
- dependency resolver 禁止未经根 marketplace 允许的跨市场传递依赖。
- MCP 工具仍经过工具 schema 和 permission 路径，但 MCP server 进程本身可能在启动时执行代码。
- command Hook 可运行进程，HTTP Hook 可访问网络；Hook 输出可改变工具决定，因此受配置来源、trust 和 policy 控制。
- LSP server 是插件声明的长期子进程，具备其 OS 账户权限；workspace/configuration 内容不能被当作安全策略。

安装时 trust 和运行时 permission 是两个边界。允许安装一个插件，不代表其所有 MCP 工具应永久自动批准。

## 14.12 凭据、日志与持久化

`src/utils/secureStorage/` 按平台选择 macOS Keychain、Windows Credential Manager、Linux secret storage，并提供 fallback。fallback 的安全等级取决于平台能力；plain-text storage 不能等同系统 keychain，敏感 token 的字段也会限制进入明文状态。

`src/utils/redaction.ts` 对 API key、Authorization、cookie、password、JWT、private key、URL userinfo/query 和 home path 脱敏；`src/utils/diagnostics/issueReport.ts` 在诊断报告输出前递归调用这些能力。诊断报告明确不收集 prompt、transcript 和文件内容。

当前 open build 的 analytics API 和 sink 是 no-op，构建还有 `no-telemetry` stub 与 `verify-no-phone-home` 检查。但以下网络流量仍属于产品功能而不是 telemetry：模型 API、OAuth、MCP、WebFetch、插件 marketplace、更新和显式 remote 功能。不能把“无产品遥测”表述成“程序绝不联网”。

会话 JSONL 默认保存完整对话和工具输入/输出，外置的大工具结果也落盘。redaction 主要保护日志和诊断出口，不会自动抹除用户要求持久化的 transcript。共享机器上必须依赖文件权限、磁盘策略和定期清理。

## 14.13 远程与 SDK 边界

Remote WebSocket 首先发送 OAuth credential，服务端握手后才建立 session。远程权限请求通过 permission bridge 回到控制端；连接断开、超时或没有审批方时必须形成拒绝/失败，而不是默认批准。

session ingress 使用独立 token，外部消息带 origin 元数据并被标记为非用户内容。身份认证解决“谁可发送”，不能解决“内容是否应被模型执行”。

Agent SDK 允许宿主提供 `canUseTool` callback、MCP server 和环境覆盖。宿主因此成为安全边界的一部分：

- callback 不应仅按工具名批准，必须检查参数和建议的 permission update。
- 同进程多 session 的环境变量覆盖由互斥量序列化，但第三方代码直接读写 `process.env` 仍是全局共享风险。
- 宿主必须消费并关闭 iterator/session，传播 AbortSignal，并保护输出落盘位置。

`src/grpc/server.ts` 是开发适配器：明文 transport、内存 session 和宽接口都不满足公网生产部署条件。不能只在其前面开放端口而不增加 TLS、认证、授权、限流和审计。

## 14.14 典型攻击链及中断点

### 恶意仓库通过配置执行命令

```text
打开仓库
  -> 项目 settings 声明 Hook/MCP/危险 env
  -> TrustDialog 阻止项目控制面提前生效
  -> 用户拒绝则退出
  -> 接受后仍由 Hook policy / tool permission / sandbox 继续约束
```

### Prompt injection 请求读取密钥并上传

```text
网页内容诱导模型
  -> FileRead 访问工作区外路径需要规则/批准
  -> WebFetch/Bash 外发需要独立权限
  -> sandbox network allowlist 可进一步阻止目标
  -> 诊断 redaction 不是此链路的防线
```

关键点是读和发是两个独立能力，但用户一次性批准过宽的 Bash 仍可能同时完成二者。

### Symlink 逃逸

```text
模型写 cwd/link/file
  -> safeResolvePath / realpath ancestor
  -> 同时检查逻辑路径和物理路径
  -> deny / dangerous path / workspace containment
  -> sandbox mount policy（若启用）
```

### MCP 工具伪装内置工具

```text
server 提供相似 tool name
  -> 外部工具保留 MCP 身份
  -> 不进入 built-in exception
  -> schema + MCP permission + 用户/宿主决定
```

## 14.15 残余风险和正确表述

| 风险 | 为什么仍存在 | 缓解方向 |
|---|---|---|
| 用户误批危险操作 | 最终批准可授予真实 OS 权限 | 窄规则、diff 预览、sandbox、policy |
| Prompt injection | 模型不能可靠区分指令与数据 | 最小工具集、独立读/写/网权限、来源标记 |
| 恶意依赖/编译脚本 | Bash 获批后可执行供应链代码 | 隔离环境、禁网、锁定依赖、人工审计 |
| TOCTOU/symlink race | 检查与打开不是统一原子操作 | OS sandbox、容器、低权限账户 |
| 本机/代理网络边界 | loopback 被允许，代理可能代解析 | 部署级 egress policy、代理 ACL |
| transcript 泄密 | 会话设计上保存完整上下文 | 文件权限、保留策略、加密磁盘 |
| 扩展供应链 | 插件/MCP/LSP 是可执行代码 | 固定来源和版本、policy、独立账户 |
| headless 对不可信仓库执行 | 无交互 trust，调用者承担信任 | CI checkout 审计、容器、禁 bypass |

面试中的准确结论应是：OpenClaude 采用多层、来源感知的 capability gating，并可叠加 OS sandbox；它显著降低模型误操作和配置注入的风险，但不把 LLM、仓库或扩展代码变成可信主体，也不承诺在用户授予任意命令后仍保持宿主机隔离。

## 14.16 源码定位

- `src/components/TrustDialog/TrustDialog.tsx`：目录信任 UI 与持久化。
- `src/utils/config.ts`：trust 查询、祖先继承和项目配置。
- `src/utils/managedEnv.ts`、`src/utils/caCertsConfig.ts`：trust 前后环境边界。
- `src/utils/permissions/`：模式、规则、路径和分类器判定。
- `src/utils/bash/`、`src/utils/shell/`：shell AST 与只读命令验证。
- `src/utils/sandbox/sandbox-adapter.ts`：sandbox 配置和平台能力。
- `src/utils/hooks/ssrfGuard.ts`：HTTP Hook SSRF 防护。
- `src/utils/secureStorage/`、`src/utils/redaction.ts`：凭据存储和脱敏。
- `src/utils/plugins/`：marketplace、安装和依赖信任。
- `src/remote/`、`src/utils/sessionIngressAuth.ts`：远程认证和会话入口。
