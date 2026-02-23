# 01 — Agent 核心概念入门

## 什么是 Agent？

**一句话定义：** Agent 是一个能自主决策并执行行动的 AI 系统。

传统的 ChatBot 像一面镜子——你说什么，它反射回什么。而 Agent 像一个有手有脚的助手——你说"帮我订明天的机票"，它会自己去搜索航班、比较价格、填写表单、完成预订。

## Agent vs ChatBot：本质区别

| 维度 | ChatBot | Agent |
|------|---------|-------|
| **核心能力** | 生成文本 | 感知→推理→行动→观察 |
| **与世界交互** | 无（只能输出文字） | 有（可以执行工具、读写文件、上网搜索） |
| **自主性** | 被动回答 | 主动规划和执行多步任务 |
| **状态** | 无状态或简单上下文 | 持久化记忆、会话管理 |
| **安全需求** | 低（只输出文字） | 高（能执行代码、操作文件系统） |

## Agent 的五大核心能力

### 1. 感知 (Perception)

Agent 需要从外部世界接收信息。在 NanoClaw 中，感知通道是 WhatsApp：

```
用户发送消息 → WhatsApp 协议 (baileys) → SQLite 存储 → 轮询检测
```

NanoClaw 的 `src/channels/whatsapp.ts` 通过 baileys 库监听 WhatsApp Web 协议，将每条消息存入 SQLite。主循环每 2 秒轮询一次数据库，检测新消息。

**关键洞察：** Agent 不需要实时响应。NanoClaw 使用轮询而非 WebSocket 推送，这让架构更简单、更可靠。

### 2. 推理 (Reasoning)

Agent 的"大脑"是大语言模型 (LLM)。NanoClaw 使用 Claude，通过 Claude Agent SDK 调用。

推理不仅仅是"回答问题"，更重要的是**决策**：
- 是否需要使用工具？用哪个？
- 是否需要搜索网络？
- 是否需要读取文件来获取更多上下文？
- 任务是否已完成，还是需要继续？

在 NanoClaw 的 `container/agent-runner/src/index.ts` 中，Claude Agent SDK 的 `query()` 函数封装了整个推理循环：

```
调用 Claude API → 检查响应 → 有工具调用？ → 执行工具 → 将结果反馈 → 再次调用 API → ...
```

这个循环会一直持续，直到 Claude 认为任务完成（响应中没有工具调用），或者达到最大轮次/预算限制。

### 3. 行动 (Action)

Agent 通过**工具 (Tools)** 与世界交互。NanoClaw 中 Agent 拥有的工具：

| 工具 | 作用 | 来源 |
|------|------|------|
| Bash | 执行 Shell 命令（在容器内，安全！） | Claude Agent SDK 内置 |
| Read/Write/Edit | 文件读写操作 | Claude Agent SDK 内置 |
| Glob/Grep | 文件搜索 | Claude Agent SDK 内置 |
| WebSearch/WebFetch | 网络搜索和获取内容 | Claude Agent SDK 内置 |
| agent-browser | 浏览器自动化（截图、表单填写等） | 容器内 Chromium |
| schedule_task | 创建定时任务 | NanoClaw MCP 服务器 |
| send_message | 发送 WhatsApp 消息 | NanoClaw MCP 服务器 |
| list_tasks/pause_task/... | 管理定时任务 | NanoClaw MCP 服务器 |

**关键洞察：** 工具的安全性至关重要。NanoClaw 让 Bash 工具在容器内执行，即使 Agent 运行了 `rm -rf /`，也只会影响容器内部，不会损害宿主机。

### 4. 记忆 (Memory)

Agent 需要记住之前的对话和用户偏好。NanoClaw 使用三层记忆系统：

```
全局记忆 (groups/global/CLAUDE.md)     ← 所有群组共享的偏好和事实
    ↓
群组记忆 (groups/{name}/CLAUDE.md)    ← 特定群组的上下文和记忆
    ↓
会话记忆 (data/sessions/{group}/)     ← Claude SDK 自动管理的对话历史
```

CLAUDE.md 文件是 Claude Agent SDK 的一个特性——当 Agent 启动时，SDK 自动加载工作目录及其父目录中的 CLAUDE.md 作为系统上下文。这意味着你只需要在一个 Markdown 文件中写入"记住我喜欢深色模式"，Agent 在每次对话中就会知道这个偏好。

**关键洞察：** 记忆不一定需要复杂的向量数据库。NanoClaw 用简单的 Markdown 文件实现了持久记忆，利用了 Claude Agent SDK 的原生机制。

### 5. 自主调度 (Autonomous Scheduling)

真正的 Agent 不只是被动等待指令，它还能自主执行任务。NanoClaw 通过任务调度器实现了这一点：

```
用户: "@Andy 每周一早上9点给我发AI新闻摘要"
                    ↓
Agent 调用 schedule_task 工具 → 创建 cron 任务
                    ↓
调度器每分钟检查到期任务 → 启动容器 → Agent 搜索新闻 → 发送消息
```

这存储在 SQLite 的 `scheduled_tasks` 表中，支持 cron 表达式（周期性）、interval（间隔）、once（一次性）三种调度类型。

## Agent 的核心循环

所有 Agent 系统的核心都是同一个循环：

```
┌──────────────────────────────────────────────┐
│                Agent 核心循环                   │
│                                                │
│  1. 接收输入（感知）                             │
│       ↓                                        │
│  2. 结合记忆和上下文，交给 LLM 推理               │
│       ↓                                        │
│  3. LLM 决定：                                  │
│     ├── 需要工具？→ 执行工具 → 获取结果 → 回到 2   │
│     └── 任务完成？→ 输出最终回复                   │
│       ↓                                        │
│  4. 更新记忆和状态                               │
│       ↓                                        │
│  5. 等待下一个输入                               │
│                                                │
└──────────────────────────────────────────────┘
```

在 NanoClaw 中：
- **步骤 1** 由 `src/index.ts` 的 `startMessageLoop()` 完成
- **步骤 2-3** 由 `container/agent-runner/src/index.ts` 的 `runQuery()` 完成
- **步骤 4** 由 SQLite 数据库和 CLAUDE.md 文件完成
- **步骤 5** 由轮询循环和空闲超时机制完成

## "Agent Harness"——为什么运行环境很重要

NanoClaw 的 README 中有一句话：

> *A bad harness makes even smart models seem dumb, a good harness gives them superpowers.*

"Harness"（运行环境/框架）是连接 LLM 和工具世界的桥梁。同样的 Claude 模型，在不同的 harness 中表现可能天差地别：

| Harness 质量 | 表现 |
|-------------|------|
| 差 | 工具调用失败率高、上下文丢失、安全漏洞 |
| 好 | 工具调用可靠、记忆持久、安全隔离、错误恢复 |

NanoClaw 使用 Claude Agent SDK（即 Claude Code 的底层 SDK）作为 harness，这意味着 Agent 拥有 Claude Code 所有的能力——文件操作、代码执行、网络搜索等——而且这些工具经过了大量的调优和测试。

## 小结

理解 Agent，只需要记住五个关键词：

1. **感知**——接收外部信息（消息、事件）
2. **推理**——LLM 思考和决策
3. **行动**——通过工具与世界交互
4. **记忆**——持久化上下文和偏好
5. **调度**——自主执行计划任务

NanoClaw 将这五个能力浓缩在约 2000 行代码中，是理解 Agent 工程的最佳起点。

---

**下一篇：** [02 — NanoClaw 架构全景解析](./02-architecture-deep-dive.md)
