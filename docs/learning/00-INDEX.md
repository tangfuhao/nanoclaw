# NanoClaw 深度学习指南 — 从 Agent 新手到核心掌握

本系列文档以 NanoClaw 为实战案例，系统性地讲解 AI Agent 的核心概念与工程实践。

## 阅读路线

| # | 文档 | 你将学到 | 预计时间 |
|---|------|---------|---------|
| 01 | [Agent 核心概念入门](./01-agent-core-concepts.md) | 什么是 Agent？与 ChatBot 的本质区别，Agent 的五大核心能力 | 15 分钟 |
| 02 | [NanoClaw 架构全景解析](./02-architecture-deep-dive.md) | 整个系统如何从一个 WhatsApp 消息变成 Agent 行为，模块间如何协作 | 20 分钟 |
| 03 | [消息流转与状态管理](./03-message-flow-and-state.md) | 消息从接收到处理到回复的完整生命周期，状态如何持久化 | 20 分钟 |
| 04 | [容器隔离与安全模型](./04-container-isolation-and-security.md) | 为什么要用容器、挂载系统设计、权限隔离、安全边界 | 15 分钟 |
| 05 | [工具系统与技能扩展](./05-tool-system-and-skills.md) | MCP 协议、IPC 通信、工具注册、技能系统的设计哲学 | 20 分钟 |
| 06 | [Claude Agent SDK 深度剖析](./06-claude-agent-sdk-internals.md) | SDK 内部机制、query() 循环、Agent Teams、Hook 系统 | 25 分钟 |
| 07 | [IPC 实现原理与完整流程](./07-ipc-deep-dive.md) | 四条 IPC 通道的工作原理、原子写入、安全保障、数据快照 | 20 分钟 |
| 08 | [NanoClaw 与 Claude Code 交互全流程](./08-nanoclaw-claude-code-interaction.md) | 从用户消息到 API 调用的完整调用链、密钥安全传递、输出回传 | 25 分钟 |
| 09 | [容器释放与上下文恢复机制](./09-session-resume-and-context.md) | 三层持久化体系、Session 恢复、CLAUDE.md 记忆、对话归档 | 20 分钟 |
| 10 | [边界情况与异常处理](./10-edge-cases-and-error-handling.md) | 并发满载排队、游标回滚、重试退避、任务优先级、宿主机重启 | 20 分钟 |

## 核心理念

NanoClaw 的设计哲学是：**小到能完全理解，安全到能放心运行**。整个代码库只有约 2000 行核心代码（不含测试），却实现了一个完整的 Agent 系统。这使它成为学习 Agent 工程的绝佳教材。

## 项目全景

```
用户 (WhatsApp) ──→ 消息存储 (SQLite) ──→ 轮询循环 ──→ 容器 (Claude Agent SDK) ──→ 回复
                                              ↑                    ↓
                                        调度器 ─────────────→ IPC 通信
```

单进程 Node.js 应用。Agent 在隔离的 Linux 容器中运行。每个群组有独立的文件系统和记忆。
