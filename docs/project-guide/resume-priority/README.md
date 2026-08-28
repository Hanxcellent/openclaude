# 简历内容优先阅读目录

本目录列出介绍三项项目经历前必须掌握的内容。建议按照表格顺序阅读。

## 第一组：核心内容

|  顺序 | 阅读内容                                                                      | 掌握目标                               |
| --: | ------------------------------------------------------------------------- | ---------------------------------- |
|   1 | [三项改进的架构位置](../18-resume-technical-deep-dives.md#181-三项改进的架构位置)           | 明确每项改进涉及的核心模块和关联模块                 |
|   2 | [会话存活判定机制](../18-resume-technical-deep-dives.md#182-会话存活判定机制)             | 掌握空闲超时、请求总时长和误终止场景                 |
|   3 | [引用计数式挂起与恢复](../18-resume-technical-deep-dives.md#183-引用计数式挂起与恢复)         | 掌握计数、时间补偿、恢复时机和异常清理                |
|   4 | [长运行工具的周期性活性上报](../18-resume-technical-deep-dives.md#184-长运行工具的周期性活性上报)   | 掌握 MCP、TaskOutput 和 Subagent 的活性传播 |
|   5 | [终端 Footer 生命周期](../18-resume-technical-deep-dives.md#185-终端-footer-生命周期) | 掌握 mounted、visible、active 三类状态     |
|   6 | [LLM 响应处理边界](../18-resume-technical-deep-dives.md#186-llm-响应处理边界)         | 掌握错误标记、共享校验、回退和失败指标                |
|   7 | [相关追问知识矩阵](../18-resume-technical-deep-dives.md#188-相关追问知识矩阵)             | 检查关键设计理由和异常路径                      |
|   8 | [三项经历的连续介绍](../18-resume-technical-deep-dives.md#189-三项经历的连续介绍)           | 形成完整口头介绍                           |

## 第二组：架构背景

| 阅读内容 | 掌握目标 |
|---|---|
| [会话执行流程中的存活判定](../04-query-agent-loop.md#16-会话存活判定) | 理解看门狗在主请求流程中的位置 |
| [Agent 长运行任务的活性传播](../08-agents-tasks-orchestration.md#17-长运行任务的活性传播) | 理解子 Agent 进度进入父会话的路径 |
| [终端 Footer 生命周期](../09-tui-interaction-flow.md#17-footer-生命周期) | 理解 Footer 与终端交互模块的关系 |
| [合法等待与真实停滞](../12-errors-retries-recovery-edge-cases.md#17-合法等待与真实停滞) | 理解超时终止和恢复规则 |
| [辅助模型请求失败](../12-errors-retries-recovery-edge-cases.md#18-辅助模型请求失败) | 理解标题失败的隔离策略 |
| [共享标题生成服务](../13-entrypoint-modes-sdk-remote.md#17-共享标题生成服务) | 理解 REPL、Remote 和 SDK 的共同调用路径 |

## 第三组：源码与测试定位

| 阅读内容 | 掌握目标 |
|---|---|
| [Agent loop 源码定位](../17-source-map-glossary.md#agent-loop) | 定位 QueryGuard、权限等待和工具活动登记 |
| [Agent、Task 与团队源码定位](../17-source-map-glossary.md#agenttask-与团队) | 定位 MCP、TaskOutput 和 Subagent 进度传播 |
| [UI 与终端源码定位](../17-source-map-glossary.md#ui-与终端) | 定位 KeepMounted、Footer 规则和 StatusLine |
| [SDK 与远程源码定位](../17-source-map-glossary.md#sdk-与远程) | 定位共享标题生成的三个调用入口 |
| [简历技术专题提交索引](../17-source-map-glossary.md#简历技术专题) | 定位五个相关提交和对应测试 |

## 阅读完成标准

完成阅读后应达到以下标准：

1. 可以用三分钟连续介绍三项改进。
2. 可以说明每项故障的现象、根因、状态变化和最终效果。
3. 可以说明引用计数、generation、活性周期和超时限制的作用。
4. 可以说明 mounted、visible、active 的职责。
5. 可以说明标题错误检查放在共享服务中的原因。
6. 可以列出允许、拒绝、中止、异常和竞争路径的清理动作。
7. 可以说明每项改进的回归测试范围。

