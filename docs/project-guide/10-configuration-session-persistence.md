# 10. 配置、会话与持久化

本章解释配置来源、对话落盘过程以及恢复、回退和分支的作用范围。OpenClaude 的主要会话存储采用带父指针和元数据事件的 JSONL 日志。

## 10.1 状态分层

| 状态 | 主要载体 | 生命周期 | 主要入口 |
|---|---|---|---|
| 用户、项目、托管配置 | 多层 `settings.json` | 跨会话 | `src/utils/settings/settings.ts` |
| 当前 UI/执行状态 | React/AppState、模块状态 | 单进程 | 第 3、9 章 |
| 主对话转录 | 每会话一个 `.jsonl` | 跨进程，可恢复 | `src/utils/sessionStorage.ts` |
| 子代理转录 | 会话目录下独立 JSONL | 随父会话 | `getAgentTranscriptPath()` |
| 文件快照 | JSONL 元数据加备份文件 | 随会话 | `src/utils/fileHistory.ts` |
| 大工具结果 | 会话目录中的 `.txt`/`.json` | 随会话 | `src/utils/toolResultStorage.ts` |
| 轻量历史 | 普通 JSON 会话文件 | 特定辅助路径 | `src/utils/sessionPersistence.ts` |

最后一行用于特定辅助路径。交互 REPL 的主转录系统位于 `sessionStorage.ts`。两者的调用点和数据格式需要分别阅读。

## 10.2 配置源及优先级

`SETTING_SOURCES` 按低到高排列：

```text
userSettings
  -> projectSettings
  -> localSettings
  -> flagSettings
  -> policySettings
```

常见路径和语义如下。

| 源 | 典型位置 | 普通设置 UI 修改权限 | 作用 |
|---|---|---:|---|
| `userSettings` | 配置根目录下 `settings.json` | 是 | 用户全局默认值 |
| `projectSettings` | `<cwd>/.openclaude/settings.json` | 是 | 可随仓库共享的项目配置 |
| `localSettings` | `<cwd>/.openclaude/settings.local.json` | 是 | 本机项目配置。写入时保证被 Git 忽略 |
| `flagSettings` | `--settings` 或 SDK 内联配置 | 否 | 本次启动覆盖 |
| `policySettings` | 托管配置、MDM 或远端策略 | 否 | 组织级强制策略，最终覆盖 |

插件提供的默认设置位于这些文件源之下。`--setting-sources` 只能限制用户、项目和本地三层。`flagSettings` 与 `policySettings` 总会加入启用集合，避免调用者绕过显式启动参数或管理员策略。

### 合并规则

配置使用 `lodash mergeWith` 深合并：

- 普通标量由后面的高优先级源覆盖。
- 对象递归合并。
- 数组合并后执行去重。
- 更新 API 用 `undefined` 表示删除字段。直接 `delete` 会绕过定制合并器的删除语义。

“最终配置”需要根据全部启用来源推导。诊断某个值时应使用带来源信息的 `getSettingsWithSources()`，并检查解析错误。

### 托管配置内部优先级

`policySettings` 在全局合并中优先级最高。其内部采用“第一个有内容的来源获胜”规则：

```text
远端托管设置
  > 系统级 MDM（Windows HKLM / macOS plist）
  > managed-settings.json + managed-settings.d/*.json
  > Windows HKCU
```

文件式管理配置内部先合并 `managed-settings.json`，再按文件名字典序合并 `managed-settings.d` 中的 JSON。文件内部采用合并规则。管理渠道之间采用单选规则。

### 读取、校验和热更新

设置加载流程为：

1. 解析 JSON。
2. 用设置 schema 校验字段。
3. 单独校验权限规则，过滤非法项并产生可显示错误。
4. 缓存每个来源的结果。
5. 按优先级合并，并保留来源和错误信息。

文件监听器检测外部修改后会清理缓存并重新应用配置。`ConfigChange` hook 可以阻止一次外部配置变更。内部写入会被标记，防止监听器把自己的写操作误判为外部变更。

危险权限提示有额外的信任限制。例如 `skipDangerousModePermissionPrompt` 不接受项目设置作为可信来源，防止恶意仓库仅通过提交配置文件就关闭关键确认。

相关源码：

- `src/utils/settings/constants.ts`
- `src/utils/settings/settings.ts`
- `src/utils/settings/changeDetector.ts`
- `src/utils/settings/applySettingsChange.ts`
- `src/utils/settings/permissionValidation.ts`
- `src/utils/settings/mdm/settings.ts`

## 10.3 主转录的目录结构

主会话目录以启动时的原始工作目录为键，并先做路径清洗：

```text
<config-projects-dir>/
  <sanitized-original-cwd>/
    <sessionId>.jsonl
    <sessionId>/
      subagents/
        agent-<agentId>.jsonl
        agent-<agentId>.meta.json
      remote-agents/
        <taskId>.json
      tool-results/
        <toolUseId>.txt
        <toolUseId>.json
```

目录按 `0700` 创建。主会话和分支文件按 `0600` 创建。恢复另一个项目或已有会话后，`sessionProjectDir` 与会话 ID 必须一起切换。单独切换其中一项会使后续消息写入错误文件。

## 10.4 JSONL 采用事件日志结构

每行是一个独立 JSON 对象。主要分两组。

### 对话链参与者

`user`、`assistant`、`attachment`、`system` 等转录消息带有：

- `uuid`：当前记录 ID。
- `parentUuid`：当前活动对话链中的直接父记录。
- `logicalParentUuid`：某些重排或压缩场景的逻辑父节点。
- `sessionId`、`cwd`、`timestamp`、`version`、`gitBranch`。
- `isSidechain`：代理旁路标记。

`tool_use` 和对应 `tool_result` 位于消息内容块中，并通过消息父链保证顺序。读取器兼容旧版 `progress` 行。旧版 `progress` 行不参与父链。加载器会桥接其前后父指针，避免旧进度记录把链截断。

### 元数据事件

同一 JSONL 还可包含：

- `summary`、`custom-title`、`ai-title`、`last-prompt`、`task-summary`、`tag`。
- `agent-name`、`agent-color`、`agent-setting`、`mode`。
- `pr-link`、`worktree-state`、`session-branch`。
- `file-history-snapshot`、`attribution-snapshot`。
- `queue-operation`、`content-replacement`、`goal-state`。
- `context-collapse-commit`、`context-collapse-snapshot`。

多数会话级元数据采用“最后一条获胜”。写入器会把当前元数据重新追加到文件尾部，使只读取头尾的会话列表也能得到最新标题、标签和模式。

## 10.5 写入协议

`recordTranscript()` 采用增量追加协议。其流程为：

1. 清洗无需记录的临时消息。
2. 查询本会话已记录 UUID 集合并去重。
3. 只把新消息交给项目级单例 writer。
4. 依据已有前缀或显式 hint 确定起始 `parentUuid`。
5. 串行追加各行，更新内存索引。

“只跟踪已记录前缀”是压缩正确性的关键。普通增长数组中，旧消息位于前缀，新消息应接在其后。压缩时新 boundary/summary 位于数组前面，后面的保留旧消息不能反向成为新 boundary 的父节点。

写请求通过队列串行化。`flushSessionStorage()` 等待队列清空。分支、退出和需要稳定快照的操作会先 flush。新会话文件延迟到首个可持久消息才物化，避免只启动后退出就留下空会话。

以下情况会跳过转录持久化：

- 测试环境，除非测试显式覆盖。
- `cleanupPeriodDays` 为 `0`。
- 启动参数关闭会话持久化。
- `CLAUDE_CODE_SKIP_PROMPT_HISTORY` 生效。

## 10.6 加载与父链重建

`loadTranscriptFile()` 先解析所有有效事件，再从叶节点沿 `parentUuid` 向根回溯，生成活动链。同一日志由此可以保留失败重试和回退后继续产生的多个分叉。旧行可以继续保留。

关键行为：

- 默认选择当前叶。需要分支列表时可保留全部叶。
- `buildConversationChain()` 只重建指定叶所属路径。
- 孤立父节点、重复 UUID、未配对工具块等情况会经过一致性检查或兼容修复。
- 解析失败的个别行不会自动变成可信消息。
- 恢复时同时返回标题、标签、工作树、目标、大结果替换决策等元数据 map。

### 大文件读取

会话列表不解析完整文件。`readSessionLite()` 只读取首尾各约 64 KiB，提取首个 prompt 和最新尾部元数据。

完整恢复对大日志提供专用路径。文件大于约 5 MiB且包含压缩边界时，`readTranscriptForLoad()` 执行单次正向分块扫描。扫描过程跳过无需加载的 attribution 行，并在最近的 compact boundary 丢弃此前累积内容。峰值内存由最终需要恢复的部分决定。

压缩段携带的保留元数据必须通过校验。若边界记录损坏，加载器采用失败关闭策略，只使用边界之后可确认的历史，不把无法验证的旧段重新注入上下文。

`MAX_TRANSCRIPT_READ_BYTES = 50 MiB` 约束需要原始整文件内容的调用者。优化加载路径只产生压缩后的有效缓冲区，因此可以处理部分超过 50 MiB 的转录文件。

## 10.7 Resume 与 Continue

`--continue` 选择最近会话。`--resume` 可以按 ID、标题或选择器指定会话。恢复过程协调以下状态：

1. 加载并检查活动父链。
2. 切换全局 session ID 和目标项目目录。
3. 恢复消息、文件历史、attribution、工具结果替换表和 collapse 状态。
4. 恢复 agent setting。若定义已不存在，则提示并降级为默认代理。
5. 未被 CLI 显式覆盖时恢复模型。
6. 恢复 normal/coordinator mode、目标、标题、标签、PR 元数据。
7. 处理已保存 worktree。
8. `adoptResumedSessionFile()` 让后续写入继续追加到原文件。

Worktree 恢复遵循明确优先级。本次新的 `--worktree` 选择具有最高优先级。没有新选择时，系统仅在保存路径存在的情况下切换目录。路径已删除时写入空 worktree 状态并回退到正常目录。中途改变 cwd 后还会清除依赖目录的 prompt、memory 和 plan 缓存。

恢复后直接退出时，`adoptResumedSessionFile()` 允许退出清理把新的 `--name` 等元数据写回已有文件。

## 10.8 对话回退与文件回退

`/rewind` 打开 `MessageSelector`，用户可选择：

| 选项 | 对话状态 | 文件状态 |
|---|---|---|
| Restore conversation | 回到目标用户消息并把原 prompt 放回输入框 | 不变 |
| Restore code | 不变 | 应用目标消息对应文件快照 |
| Restore code and conversation | 两者都执行 | 两者都执行 |
| Summarize from here | 生成部分压缩边界和摘要 | 不变 |

两个恢复操作各自捕获错误。系统会分别报告文件恢复结果和对话恢复结果。只有两项成功时，整个操作才会报告成功。

对话回退通过内存父链和消息集裁剪实现。旧 JSONL 记录通常保留为非活动分支。失败流式尝试留下的孤立消息采用 `removeTranscriptMessage()` 删除：

1. 常见情况只扫描文件尾部 64 KiB，定位精确 `\"uuid\":\"...\"` 字段并原位截断/搬移尾行。
2. 找不到时才整文件读取和重写。
3. 慢路径遇到超过 50 MiB 的文件会跳过删除，以避免 OOM。恢复父链负责排除该孤立行。

文件回退由 `fileHistoryRewind()` 应用目标 snapshot。该操作产生文件系统副作用。`FileHistoryState` 本身保持当前状态。不存在快照、文件写入失败或删除失败都会抛出错误。该功能基于 OpenClaude 文件历史快照。Git 状态和未被文件历史跟踪的外部变更不在其恢复范围内。

## 10.9 Conversation Branch

`/branch [name]` 的正确语义是复制对话链：

```text
当前 JSONL --flush--> 选择当前活动 leaf
                    -> 沿 parentUuid 重建唯一活动链
                    -> 改写 sessionId 和连续 parentUuid
                    -> 复制必要会话元数据与替换决策
                    -> 写入 <newSessionId>.jsonl
                    -> 切换到新会话
```

分支元数据记录直接父会话、根会话、来源会话、分支时间和分支点消息 ID。多层分支保留最初的 `rootSessionId`。

分支操作遵循以下规则：

- 原会话文件不修改。
- 只复制当前活动父链，不复制其他 leaf。
- 新文件保留消息时间、Git 分支等消息元数据。会话 ID 会被重写。
- `content-replacement` 决策一并复制，避免恢复后突然把原始大结果放回 prompt，破坏缓存和 token 预算。
- 自动切换失败时新文件保持有效。用户可按新 ID 手动恢复。
- 这是 conversation branching，不创建 worktree，也不隔离文件系统。

## 10.10 大工具结果的稳定外置

每个工具声明 `maxResultSizeChars`。超过有效阈值的纯文本结果会写入：

```text
<sessionDir>/tool-results/<toolUseId>.txt|json
```

模型上下文中的原结果被替换为约 2,000 字节预览和文件路径。普通文本采用 UTF-8 安全的 head-tail。序列化 JSON 采用 head-only。预览会明确标注片段可能不构成有效 JSON。该标注可以避免尾部拼接产生错误对象。非文本块保留在消息内。写盘失败时保留原结果。

每条消息还有聚合预算。系统优先外置最大的新增结果。某个 tool result 的处理决定形成后，会通过 `content-replacement` 记录冻结：

- 已经保留完整的旧结果不会在后续轮次突然变为预览。
- 已经替换的结果在 resume、branch 和 prompt replay 后使用相同表示。
- REPL 同步替换内存转录中的内容，以释放堆内存。
- `maxResultSizeChars = Infinity` 表示工具自行控制结果，不进入外置循环。

这种确定性主要服务于 prompt cache：同一历史在后续轮次必须保持字节表示稳定。

## 10.11 关键不变量

1. 会话 ID、写入文件路径和恢复项目目录必须一致切换。
2. 所有持久消息 UUID 在同一会话内去重。
3. 父指针只指向链参与者，不指向临时 progress。
4. 压缩 boundary 必须成为新活动链的根或受验证连接点。
5. 工具结果替换决策一旦记录，恢复时必须重放。
6. 对话分支不能被描述为文件系统隔离。
7. 文件回退和对话回退是独立事务，UI 必须显示部分失败。
8. 托管策略不能被项目配置或 `--setting-sources` 排除。

## 10.12 源码阅读顺序

1. `src/utils/settings/constants.ts`
2. `src/utils/settings/settings.ts`
3. `src/types/logs.ts`
4. `src/utils/sessionStoragePortable.ts`
5. `src/utils/sessionStorage.ts`
6. `src/utils/sessionRestore.ts`
7. `src/utils/fileHistory.ts`
8. `src/utils/toolResultStorage.ts`
9. `src/components/MessageSelector.tsx`
10. `src/commands/branch/branch.ts`

下一章进入扩展系统，说明 MCP、插件、hooks、LSP 与 slash commands 对工具、上下文和控制流的扩展。
