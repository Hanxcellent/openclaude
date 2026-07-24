# 05. 上下文、Prompt、记忆与压缩

## 1. 模型实际接收的内容

一次请求并非简单拼接历史。概念上包含：

```text
system prompt
+ system context（环境/能力/日期/cwd 等）
+ user context（CLAUDE.md、规则、用户偏好）
+ active messages after compact boundary
+ turn attachments（IDE、git、todos、memory、skills 等）
+ tool definitions filtered for this request
```

最终由 API adapter 转成目标 provider 的 role/content/tool schema。

## 2. System Prompt 构造

`src/context.ts`、`src/context/` 和工具 prompt 共同构造系统提示。主要内容：

- Code Agent 身份和行为约束；
- 可用工具及调用规则；
- 文件/命令安全说明；
- 环境信息和 working directories；
- permission mode 特定规则；
- team/coordinator/assistant addendum；
- custom `--system-prompt` 或 append prompt；
- output style。

工具 description 是 API tool schema 的一部分；动态变化会破坏 provider prompt cache。因此 AgentTool 支持把动态 agent list 作为 message attachment，而工具 description 保持稳定。

## 3. 用户与项目指令

项目会发现不同层级的指令文件及 include：

- 用户级规则；
- 项目级 `CLAUDE.md` / 兼容项目指令；
- local/private rules；
- managed rules；
- `--add-dir` 对应目录指令。

重要边界：

- 项目指令在 trust 前不能触发可执行扩展；
- include/嵌套遍历有路径和循环保护；
- read-only Explore/Plan 子代理可省略 ClaudeMd，减少无关 token；
- compaction 后需要重新附上仍有效的项目规则。

Hooks 的 `InstructionsLoaded` 事件可观察指令来源和加载原因，但不应绕过 trust。

## 4. Attachments 的懒注入

`getAttachmentMessages()` 按当前轮和上下文状态生成附件。典型策略：

- 第一个 queued command 注入一次 turn-level attachment，后续同批命令不重复；
- memory 预取与模型请求前准备并行，未完成时不阻塞，可在后续循环消费；
- 已被 Read/Write/Edit 触达的 memory/file 不重复注入；
- 相同 memory attachment 去重；
- 计划模式退出、auto mode 退出只在需要时加一次说明；
- background agent/async task 状态在 compact 后重建。

## 5. 文件上下文与 Read cache

ToolUseContext 带 `readFileState`：记录模型最近读取的文件内容和时间。用途：

- Edit/Write 前的 read-before-write 语义；
- compaction 后选择最近文件恢复；
- 避免重复 memory 注入；
- 子代理 fork 时 clone 父 read cache；独立子代理则用有大小上限的新 cache。

压缩后最多恢复有限文件，每文件和总 token 都有预算，防止刚压缩完又把大文件全部塞回。

## 6. Skills 与 Commands

Skill 通常是 Markdown 指令，可来自：

- 用户/项目 skills 目录；
- 插件；
- bundled skills；
- MCP skills（feature 下）。

调用 skill 时：

1. slash command 解析其 frontmatter；
2. 可改变 allowed tools、model、effort；
3. 内容作为消息/附件进入上下文；
4. `bootstrap/state` 记录 invoked skill；
5. compaction 后以有上限的 attachment 恢复；
6. 子代理结束时清理 scoped skill，避免全局 map 无界增长。

插件 skill 名字可 namespaced；Agent frontmatter 的 bare skill name 通过 exact、plugin prefix、suffix 三步解析。

## 7. Token 估算与警告

`tokenCountWithEstimation()` 优先使用最近 assistant usage；没有可靠 usage 时估算。需要区别：

- input usage 通常是该请求完整上下文；
- output usage 是该轮输出；
- cache creation/read token 仍属于输入规模；
- snip 移除消息后，旧 usage 可能高估，需要减去已知 freed tokens。

UI 用 `calculateTokenWarningState()` 展示临近上下文上限；query 还会做 blocking guard，避免在 auto-compact breaker 激活时继续发送必然过大的请求。

## 8. Auto Compact

`services/compact/autoCompact.ts`：

- 有效上下文窗口 = 模型 context window 减预留 summary output；
- 默认在窗口下方保留 30k token buffer；小窗口使用安全 floor；
- 可由 memory pressure 强制触发；
- compact/session-memory 子查询有递归 guard；
- 连续失败 3 次触发 circuit breaker；
- breaker cooldown 默认 5 分钟，之后 half-open 再试。

跟踪 state 包括 turnId、是否已 compact、连续失败、下次 retry 时间和 force reason，必须随 query state 传递。

## 9. Compact 过程

`compactConversation()` 的步骤：

1. 判断有足够消息；
2. 执行 PreCompact hooks，合并用户与 Hook 指令；
3. 去掉图片和压缩后会重注入的附件，减少 summary 请求；
4. 调模型生成结构化 conversation summary；
5. 遇到 compaction 自身 prompt-too-long，按 API round group 从头截断，最多重试；
6. 创建 compact boundary；
7. 保留必要的 recent messages；
8. 创建文件、plan、skill、plan-mode、async-agent 等恢复附件；
9. 执行 PostCompact hooks；
10. 写 transcript boundary 和 metadata。

重建顺序固定：

```text
boundary → summary messages → preserved recent messages
→ restored attachments → hook results
```

顺序变化会影响 parent chain 和 prompt cache。

## 10. Compact Boundary 与持久化

Boundary 表示旧上下文不再进入 active chain。若保留了边界前的 recent messages，boundary 会记录 preserved segment anchor；resume 时 `applyPreservedSegmentRelinks()` 把这些物理上位于边界前的消息接到 summary 后。

失败时 fail closed：若 relink 无法证明安全，不应随意把孤立旧消息接入主链。

## 11. Reactive Compact 和 Context Collapse

### 11.1 Context Collapse

可先把已 staged 的大上下文片段提交为较小表示。遇到真实 413/prompt-too-long 时，先 drain staged collapse；这比全局总结保留更多粒度。

### 11.2 Reactive Compact

当 provider 明确拒绝上下文或媒体大小时：

- 只尝试一次；
- 对 media error，压缩输入时会剥离图片；
- 成功后 yield 新 boundary/summary 并重试原 turn；
- 若 preserved tail 仍含超大 media，第二次错误直接透出，防止循环。

### 11.3 Microcompact / Snip / Tool result replacement

- microcompact 删除或缓存旧 tool content，生成边界说明；
- snip 显式移除部分消息，恢复时应用 removal；
- 大工具结果写入磁盘，消息中保留 preview；
- content replacement 决策写 JSONL，resume/sidechain 需要重放相同替换以保持 prompt cache 前缀稳定。

## 12. Prompt Cache

缓存有效依赖请求前缀稳定：

- system prompt 和 tool schema 不应无故动态变化；
- agent list 从 tool description 移到 message 可减少 schema cache bust；
- compact 后 cache 前缀自然改变；
- fallback 到不同 provider/model 时 thinking 与 cache 前缀不一定可复用；
- content replacement 在 resume 时必须一致；
- Anthropic native compatible 路径才适合共享某些 compaction cache。

## 13. 特殊恢复说明

- provider 报 context window exceeded：强制 compact 并重试一次；
- compact 无法缩小：给用户 `/compact`、撤销大输出或 `/new` 建议；
- auto-compact breaker 冷却中且已超阈值：在发请求前停止，避免 storm；
- max output token：先提高 cap（gate 下），再用 meta nudge 续写，最多三次；
- API task budget 在 compact 后扣除已消耗上下文，不能被压缩“重置”。

下一章：[06 模型供应商与协议适配](06-provider-model-transport.md)。
