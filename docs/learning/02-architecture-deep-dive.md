# 02 — NanoClaw 架构全景解析

## 一句话概括

NanoClaw 是一个**单进程 Node.js 应用**，它监听 WhatsApp 消息，将触发消息路由到隔离的 Linux 容器中运行 Claude Agent SDK，并将 Agent 的回复发送回 WhatsApp。

## 架构全景图

```
┌────────────────────────────────────────────────────────────────────────┐
│                           宿主机 (macOS/Linux)                          │
│                         单个 Node.js 进程                               │
│                                                                        │
│  ┌──────────────┐                        ┌────────────────────┐        │
│  │  WhatsApp    │ ←─ baileys 库 ──────→  │   SQLite 数据库     │        │
│  │  消息通道     │                        │   (messages.db)    │        │
│  └──────────────┘                        └────────┬───────────┘        │
│                                                   │                    │
│         ┌─────────────────────────────────────────┘                    │
│         │                                                              │
│         ▼                                                              │
│  ┌──────────────────┐   ┌──────────────────┐   ┌─────────────────┐    │
│  │   消息循环        │   │   调度器循环      │   │   IPC 监视器     │    │
│  │  每2秒轮询DB     │   │  每60秒检查任务   │   │  每1秒扫描文件   │    │
│  └───────┬──────────┘   └───────┬──────────┘   └────────┬────────┘    │
│          │                      │                       │              │
│          └──────────┬───────────┘                       │              │
│                     │ 通过 GroupQueue 调度               │              │
│                     ▼                                   │              │
│  ┌──────────────────────────────────────────────────────┼──────────┐   │
│  │                    容器管理层                          │          │   │
│  │  container-runner.ts ← 构建挂载 + 启动容器             │          │   │
│  │  container-runtime.ts ← Docker/Apple Container 抽象   │          │   │
│  │  group-queue.ts ← 并发控制（最多5个容器同时运行）       │          │   │
│  └──────────────────────────┬───────────────────────────┘          │   │
│                              │                                     │   │
├──────────────────────────────┼─────────────────────────────────────┤   │
│                  容器边界 ════╪════                                 │   │
│                              ▼                                     │   │
│  ┌────────────────────────────────────────────────────────────┐    │   │
│  │                    Linux 容器                               │    │   │
│  │                                                            │    │   │
│  │  ┌────────────────────────────────────────────────────┐    │    │   │
│  │  │              Agent Runner (index.ts)                │    │    │   │
│  │  │                                                    │    │    │   │
│  │  │  1. 从 stdin 读取 JSON 输入                         │    │    │   │
│  │  │  2. 调用 Claude Agent SDK query()                  │    │    │   │
│  │  │  3. 流式输出结果到 stdout                           │    │    │   │
│  │  │  4. 等待 IPC 输入或关闭信号                          │    │    │   │
│  │  │  5. 循环：有新消息 → 再次调用 query()                │    │    │   │
│  │  │                                                    │    │    │   │
│  │  │  工具: Bash, Read/Write, WebSearch, agent-browser   │    │    │   │
│  │  │  MCP:  nanoclaw (schedule_task, send_message, ...)  │    │    │   │
│  │  └────────────────────────────────────────────────────┘    │    │   │
│  │                                                            │    │   │
│  │  挂载点:                                                   │    │   │
│  │   /workspace/group  ← groups/{name}/  (读写)               │    │   │
│  │   /workspace/global ← groups/global/  (只读，非main群组)     │    │   │
│  │   /home/node/.claude ← data/sessions/{group}/.claude/      │    │   │
│  │   /workspace/ipc    ← data/ipc/{group}/                    │    │   │
│  │   /workspace/extra/* ← 额外挂载目录                         │    │   │
│  └────────────────────────────────────────────────────────────┘    │   │
└────────────────────────────────────────────────────────────────────────┘
```

## 核心模块解析

### 1. 入口编排器 — `src/index.ts`

这是整个系统的"大脑"。它负责：

- **初始化**：启动容器运行时 → 初始化数据库 → 加载状态 → 连接 WhatsApp
- **消息循环**：每 2 秒轮询 SQLite，检测新消息，分发给对应群组
- **Agent 调用**：通过 `runAgent()` 函数启动容器化的 Agent
- **状态管理**：跟踪每个群组的最后处理时间戳、会话 ID

核心流程（简化版）：

```typescript
// 1. 检测新消息
const { messages } = getNewMessages(jids, lastTimestamp, ASSISTANT_NAME);

// 2. 按群组分组
const messagesByGroup = new Map<string, NewMessage[]>();
for (const msg of messages) { /* 分组 */ }

// 3. 检查触发词（非 main 群组需要 @Andy 前缀）
const hasTrigger = groupMessages.some(m => TRIGGER_PATTERN.test(m.content.trim()));

// 4. 尝试将消息发送给活跃容器，或排队启动新容器
if (queue.sendMessage(chatJid, formatted)) {
    // 消息管道发送给运行中的容器
} else {
    queue.enqueueMessageCheck(chatJid); // 排队启动新容器
}
```

### 2. 消息通道 — `src/channels/whatsapp.ts`

WhatsApp 通道实现了 `Channel` 接口，这是一个简洁的抽象层：

```typescript
interface Channel {
    name: string;
    connect(): Promise<void>;
    sendMessage(jid: string, text: string): Promise<void>;
    isConnected(): boolean;
    ownsJid(jid: string): boolean;
    disconnect(): Promise<void>;
    setTyping?(jid: string, isTyping: boolean): Promise<void>;
}
```

**设计亮点：** 通过 `Channel` 接口抽象，NanoClaw 可以轻松扩展到 Telegram、Discord 等其他平台。每个通道只需要实现这几个方法。`ownsJid()` 方法决定哪个通道负责处理特定的聊天 ID。

WhatsApp 通道的关键行为：
- 使用 baileys 库连接 WhatsApp Web
- 断线自动重连（除非是主动登出）
- 消息发送队列：断线时缓存消息，重连后自动发送
- 群组元数据每 24 小时同步一次

### 3. 消息路由 — `src/router.ts`

路由模块极度简洁（仅 44 行），但功能精确：

```typescript
// 将消息格式化为 XML（LLM 友好格式）
function formatMessages(messages: NewMessage[]): string {
    return `<messages>\n${lines.join('\n')}\n</messages>`;
}

// 输出格式：去除 <internal> 标签
function formatOutbound(rawText: string): string {
    return text.replace(/<internal>[\s\S]*?<\/internal>/g, '').trim();
}
```

**关键设计决策：**
- 消息用 XML 格式传递给 LLM（`<message sender="John" time="...">`），因为 LLM 对 XML 的理解比纯文本更精确
- Agent 可以用 `<internal>` 标签包裹内部推理，这些内容不会发送给用户

### 4. 容器运行器 — `src/container-runner.ts`

这是连接宿主机和容器世界的桥梁。核心职责：

**构建挂载卷：**
```
群组文件夹     → /workspace/group      (读写)
全局记忆       → /workspace/global     (只读，非main)
Claude会话     → /home/node/.claude    (读写)
IPC目录        → /workspace/ipc        (读写)
Agent Runner源码 → /app/src            (读写)
额外目录       → /workspace/extra/*    (按配置)
```

**通信协议：**
```
宿主机 ──stdin (JSON)──→ 容器
       ←─stdout (标记对)─ 容器

输出标记对：
---NANOCLAW_OUTPUT_START---
{"status": "success", "result": "回复文本", "newSessionId": "..."}
---NANOCLAW_OUTPUT_END---
```

**为什么用标记对而不是直接解析 JSON？** 因为容器内的 Claude Agent SDK 会在 stdout 中输出大量调试信息（工具调用日志等），标记对让宿主机能从噪音中精确提取结构化输出。

### 5. 并发控制 — `src/group-queue.ts`

`GroupQueue` 类实现了一个精巧的并发控制系统：

```
┌─────────────────────────────────────────────────────┐
│                  GroupQueue                           │
│                                                     │
│  全局并发限制: MAX_CONCURRENT_CONTAINERS = 5          │
│                                                     │
│  每个群组独立状态:                                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐            │
│  │ Group A  │ │ Group B  │ │ Group C  │            │
│  │ active   │ │ waiting  │ │ active   │            │
│  │ 2 tasks  │ │ 1 msg    │ │ 0 tasks  │            │
│  └──────────┘ └──────────┘ └──────────┘            │
│                                                     │
│  调度策略:                                           │
│  1. 同一群组的任务串行执行                              │
│  2. 不同群组的任务并行执行（受全局限制）                  │
│  3. 任务优先于消息（任务不可从 DB 重新发现）              │
│  4. 容器空闲时可接收新消息（管道模式）                    │
│  5. 失败指数退避重试（最多 5 次）                       │
│                                                     │
│  管道模式 (Pipeline):                                 │
│  活跃容器通过 IPC 文件接收后续消息                       │
│  → 避免为每条消息启动新容器                              │
│  → 保持对话连贯性                                      │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**关键创新：管道模式 (Pipeline)**

传统方式：每条消息 → 启动容器 → 处理 → 销毁容器。

NanoClaw 的方式：第一条消息 → 启动容器 → 处理 → **保持容器活跃** → 后续消息通过 IPC 文件传入 → 在同一会话中继续对话 → 空闲 30 分钟后销毁。

这带来两大好处：
1. **低延迟**：后续消息无需等待容器启动（约 5-10 秒）
2. **上下文连贯**：同一次对话在同一个 Claude 会话中进行

### 6. 数据库层 — `src/db.ts`

使用 `better-sqlite3`（同步 SQLite 驱动），包含 7 个表：

| 表 | 用途 |
|---|------|
| `chats` | 聊天元数据（JID、名称、最后消息时间） |
| `messages` | 消息内容（ID、发送者、内容、时间戳） |
| `scheduled_tasks` | 定时任务定义 |
| `task_run_logs` | 任务执行历史 |
| `router_state` | 路由器状态（键值对） |
| `sessions` | 群组会话 ID |
| `registered_groups` | 已注册群组信息 |

**为什么用 SQLite 而不是 PostgreSQL？** 单用户系统不需要网络数据库。SQLite 零配置、零维护，且使用同步 API 让代码更简单。

### 7. IPC 通信 — `src/ipc.ts`

容器内的 Agent 需要与宿主机通信（发消息、创建任务等）。NanoClaw 使用**基于文件的 IPC**：

```
容器 → 写 JSON 文件到 /workspace/ipc/messages/*.json → 宿主机轮询读取 → 执行操作
容器 → 写 JSON 文件到 /workspace/ipc/tasks/*.json   → 宿主机轮询读取 → 执行操作
宿主机 → 写消息到 data/ipc/{group}/input/*.json      → 容器轮询读取 → 注入会话
宿主机 → 写 _close 哨兵文件                          → 容器检测到后优雅退出
```

**为什么用文件而不是 HTTP/gRPC？** 
1. 容器的文件挂载是现成的通信通道
2. 原子性：先写临时文件，再 rename（保证不会读到半写的 JSON）
3. 简单：不需要额外的网络配置
4. 可审计：IPC 文件可以保留用于调试

**安全考量：** 每个群组有独立的 IPC 目录（`data/ipc/{group}/`），宿主机根据目录路径确定消息来源群组的身份。非 main 群组不能向其他群组发消息或创建任务。

### 8. 任务调度器 — `src/task-scheduler.ts`

调度器每 60 秒检查一次到期任务，找到后通过 `GroupQueue` 排队执行：

```typescript
const dueTasks = getDueTasks(); // SELECT * WHERE status='active' AND next_run <= NOW()
for (const task of dueTasks) {
    queue.enqueueTask(chatJid, taskId, () => runTask(task, deps));
}
```

任务执行后自动计算下次运行时间：
- `cron`：解析 cron 表达式计算下一次
- `interval`：当前时间 + 间隔毫秒数
- `once`：标记为 completed

## 模块依赖关系

```
index.ts (编排器)
├── channels/whatsapp.ts  → 消息收发
├── db.ts                 → 数据持久化
├── router.ts             → 消息格式化
├── container-runner.ts   → 容器生命周期
│   ├── container-runtime.ts  → Docker/Apple Container 抽象
│   └── mount-security.ts     → 挂载安全校验
├── group-queue.ts        → 并发控制
├── ipc.ts                → 容器通信
├── task-scheduler.ts     → 定时任务
├── config.ts             → 配置常量
├── types.ts              → TypeScript 类型
└── logger.ts             → 日志
```

## 关键设计模式

### 1. 文件即配置 (File as Config)

NanoClaw 不使用 YAML/TOML 配置文件。配置直接写在代码中（`src/config.ts`），用户通过让 AI 修改代码来自定义。这符合其"AI 原生"的哲学——代码库小到 AI 可以安全修改。

### 2. 技能而非功能 (Skills over Features)

不往核心代码添加功能，而是通过 `.claude/skills/` 目录中的技能文件教 Claude Code 如何转换代码库。例如 `/add-telegram` 技能不会添加一个 Telegram 模块并列于 WhatsApp 旁边，而是教 Claude 如何将整个通道替换为 Telegram。

### 3. 隔离优于权限 (Isolation over Permission)

不用应用层的权限检查（"这个 Agent 不能访问 /etc"），而是用操作系统级别的容器隔离（容器里根本看不到 /etc）。

### 4. 最小化 + AI 辅助

代码保持极简，依赖 AI 来补充文档、调试和自定义。这是一种新的软件设计范式——代码不需要"自文档化"，因为 AI 始终在场。

## 小结

NanoClaw 的架构可以用一句话概括：

> **单进程轮询消息 → 按群组排队 → 在隔离容器中运行 Claude Agent SDK → 通过文件 IPC 通信 → 将回复发送回通道。**

每个模块都极度简洁，整个系统没有微服务、没有消息队列、没有抽象层。这种极简主义使得即使是 Agent 新手也能在 8 分钟内理解整个系统。

---

**下一篇：** [03 — 消息流转与状态管理](./03-message-flow-and-state.md)
