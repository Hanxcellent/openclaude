# 13. 使用入口与部署形态

## 1. 入口设计

OpenClaude 提供多种使用入口。各入口共用会话编排、上下文、模型、工具和持久化模块。输入输出、权限确认和资源生命周期由入口单独管理。

| 入口 | 主要用户 | 输入 | 输出 |
|---|---|---|---|
| TUI | 终端用户 | 键盘、粘贴、附件 | 交互界面 |
| Headless | Shell 脚本和自动化 | 参数、stdin、结构化消息 | 文本或 JSON |
| SDK | 应用开发者 | API 调用 | 异步事件 |
| MCP Server | MCP Client | tools/list 和 tools/call | MCP 结果 |
| Remote | 远程控制端 | WebSocket 消息 | 远程界面事件 |
| SSH | 远端开发环境 | SSH 命令和转发 | 远端会话 |
| Background CLI | 长时间命令 | CLI 参数 | 日志和 attach |
| gRPC | 开发实验 | 双向 stream | 结构化事件 |

## 2. TUI

TUI 提供完整本地交互能力：

- 多轮对话。
- 流式文本和工具进度。
- 权限确认对话框。
- 模型、配置和会话选择。
- Agent 与任务视图。
- 滚动、搜索和 Fullscreen。
- 本地交互命令。

TUI 在启动时执行目录信任确认。项目扩展在确认后加载。

## 3. Headless

Headless 适用于脚本、CI 和批量任务。它不加载 React/Ink 界面，并保留相同的会话与工具流程。

### 3.1 输出形式

| 形式 | 内容 |
|---|---|
| Text | 最终文本回答 |
| JSON | 最终结构化结果 |
| Stream JSON | 模型、工具、Hook 和任务事件流 |

诊断信息与机器可解析输出使用不同通道。

### 3.2 输入形式

Headless 可以从命令参数、stdin 或双向结构化协议接收输入。双向协议支持运行中消息、权限响应和中止控制。

### 3.3 权限

Headless 没有本地权限对话框。操作需要预设规则、Permission Hook 或外部权限回调。无法获得决定的 Ask 会转为拒绝。

调用方负责确认工作目录来源。

## 4. Bare 模式

Bare 模式减少自动加载能力。它可以跳过 Hook、LSP、插件同步、自动记忆、后台预取和项目指令自动发现。

调用方明确提供的 model、MCP、Agent、Skill、settings 和附加目录继续生效。Bare 适用于受控脚本和诊断场景。

## 5. Structured I/O

Structured I/O 将会话事件编码为稳定消息格式。主要事件包括：

- Session 初始化。
- Assistant 文本和工具请求。
- 工具进度和结果。
- Permission 请求和响应。
- Hook 事件。
- Task 通知。
- 中止和错误。

每个控制请求具有相关 ID。调用方可以将响应与原请求对应。

## 6. SDK V1

SDK V1 提供一次查询接口。调用方提交 Prompt 和配置，并通过异步迭代器读取事件。

SDK 支持：

- System Prompt 追加或替换。
- Model 和 provider 设置。
- Agent 和 Skill 注入。
- MCP Server。
- Permission callback。
- Continue、Resume 和 Fork。
- 工具调用上限和 token 预算。

调用方需要持续消费迭代器，并传播中止信号。

## 7. SDK 会话隔离

同一进程可以创建多个 SDK 会话。每个会话具有独立 session ID、工作目录、消息历史和权限回调。

部分 provider 配置通过进程环境传递。系统在修改环境期间使用互斥控制，并在请求结束后恢复原值。大量使用不同环境覆盖的 SDK 请求可能串行执行。

## 8. SDK V2 持久会话

SDK V2 提供可以多次发送消息的 Session 对象。

主要操作包括：

- `sendMessage` 提交新输入。
- `stream` 读取事件。
- `interrupt` 中止当前请求。
- `respondToPermission` 回应权限请求。
- `close` 释放会话资源。

Close 会释放 MCP client、等待中的权限请求、计时器、队列和会话引擎。调用应用需要在结束使用时关闭 Session。

## 9. Session Management

SDK 可以在不启动模型的情况下执行会话管理：

- 列出会话。
- 读取会话信息和消息。
- 重命名和添加标签。
- 删除会话。
- 创建分支。

Session ID 需要通过格式校验。读取消息时，系统沿 parent UUID 重建当前消息链。修改元数据时，调用方需要避免与正在运行的同一会话并发写入。

## 10. OpenClaude 作为 MCP Server

MCP Server 入口通过 stdio 提供工具列表和工具调用。外部 MCP Client 可以调用 OpenClaude 暴露的工具。

该入口聚焦工具服务。它不提供完整对话历史、终端权限界面和全部本地命令。工具执行继续经过配置和权限检查。

## 11. Remote

Remote 模式将输入、显示事件和权限请求通过远程连接传输。实际 QueryEngine 所在进程负责模型和工具执行。

远程执行端拥有自己的文件系统、MCP 连接和权限状态。查看端只展示远端已经产生的事件，不会重复执行工具。

连接断开后，系统显示重连状态。重连超过限制时进入正常关闭流程。

## 12. SSH

SSH 入口在远端主机启动 OpenClaude，并建立输入输出转发。远端环境负责文件、Shell、模型凭据和工具执行。

本地端负责终端交互和连接管理。认证、远端命令和 socket 位置受到独立检查。

## 13. Background CLI

Background CLI 创建独立 Headless 子进程，并将进程 ID、session ID、命令和日志位置写入注册表。

用户可以：

- 查看后台会话。
- 读取日志。
- Attach 到运行会话。
- 停止进程。

后台进程保留独立生命周期。父 Shell 退出不会直接结束它。

## 14. gRPC

gRPC 入口用于开发实验。它提供双向 Chat stream 和内存会话。

当前实现使用明文连接和有限访问控制。公网生产部署需要增加 TLS、认证、授权、限流、持久会话、审计和多租户隔离。

## 15. 能力矩阵

| 能力 | TUI | Headless | SDK | MCP Server | Remote |
|---|---:|---:|---:|---:|---:|
| 多轮会话 | 是 | 取决于协议 | 是 | 否 | 是 |
| 流式事件 | 是 | 是 | 是 | 工具级 | 是 |
| 本地权限界面 | 是 | 否 | 否 | 否 | 远程桥接 |
| Session 恢复 | 是 | 是 | 是 | 否 | 是 |
| Local Interactive Command | 是 | 否 | 否 | 否 | 否 |
| MCP Client | 是 | 是 | 是 | 当前范围有限 | 取决于执行端 |
| Agent 定义 | 是 | 是 | 是 | 当前范围有限 | 取决于执行端 |
| 工具执行 | 本地 | 本地 | SDK 进程 | Server 进程 | 执行端进程 |

## 16. 入口选择

- 日常开发使用 TUI。
- 脚本和 CI 使用 Headless。
- 应用内嵌使用 SDK。
- 向 MCP Client 提供工具使用 MCP Server。
- 远程操作现有环境使用 Remote 或 SSH。
- 长时间无人值守任务使用 Background CLI。
- gRPC 仅用于开发验证。

下一章说明所有入口共用的安全检查和残余风险。
