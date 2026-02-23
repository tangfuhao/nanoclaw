# 07 — IPC 实现原理与完整流程

## 为什么需要 IPC？

容器内的 Agent 运行在一个隔离的 Linux 环境中，它**没有网络端口**、**不能直接访问宿主机 SQLite**、**不能直接调用 WhatsApp 库发消息**。但 Agent 需要：
- 给用户发消息
- 创建/管理定时任务
- 注册新群组

解决方案是 **基于文件系统的 IPC (Inter-Process Communication)**——容器和宿主机共享一个目录，双方通过在这个目录中读写 JSON 文件来通信。

## IPC 的物理结构

```
宿主机路径: data/ipc/{group}/
容器内路径: /workspace/ipc/

data/ipc/
├── main/                          ← main 群组的 IPC 命名空间
│   ├── messages/                  ← 容器 → 宿主机：待发送的消息
│   │   └── 1708688400000-a3f2.json
│   ├── tasks/                     ← 容器 → 宿主机：任务操作请求
│   │   └── 1708688401000-b7c1.json
│   ├── input/                     ← 宿主机 → 容器：后续消息注入
│   │   ├── 1708688402000-d9e3.json
│   │   └── _close                 ← 关闭哨兵文件
│   ├── current_tasks.json         ← 宿主机 → 容器：任务快照（只读）
│   └── available_groups.json      ← 宿主机 → 容器：群组列表快照
│
├── family-chat/                   ← family-chat 群组的 IPC 命名空间
│   ├── messages/
│   ├── tasks/
│   ├── input/
│   └── current_tasks.json
│
└── errors/                        ← 处理失败的 IPC 文件（调试用）
    └── family-chat-1708688400000-x.json
```

**每个群组有独立的 IPC 命名空间**——这是安全模型的核心。宿主机通过 IPC 文件所在的**目录路径**来确定消息来源群组的身份，容器无法伪造。

## 四条 IPC 通道

### 通道 1：容器 → 宿主机（发消息）

```
场景: Agent 调用 send_message 工具

┌─────────────────────────────────────────────────────────────┐
│ 容器内                                                       │
│                                                             │
│ 1. Claude 调用工具: mcp__nanoclaw__send_message              │
│    参数: { text: "天气晴朗，12°C" }                           │
│         ↓                                                    │
│ 2. MCP Server (ipc-mcp-stdio.ts) 收到调用                    │
│         ↓                                                    │
│ 3. writeIpcFile('/workspace/ipc/messages/', {                │
│      type: 'message',                                        │
│      chatJid: '120363...@g.us',  ← 从环境变量注入             │
│      text: '天气晴朗，12°C',                                  │
│      groupFolder: 'family-chat', ← 从环境变量注入             │
│      timestamp: '2026-02-23T10:30:00Z'                       │
│    })                                                        │
│         ↓                                                    │
│ 4. 原子写入:                                                  │
│    a. 写 .tmp 临时文件                                        │
│    b. rename 为 .json（原子操作）                              │
│                                                             │
│ 5. 返回给 Claude: "Message sent."                            │
└─────────────────────────────────────────────────────────────┘
         ↓  (文件在共享挂载卷上，宿主机立即可见)
┌─────────────────────────────────────────────────────────────┐
│ 宿主机                                                       │
│                                                             │
│ 6. IPC Watcher 每 1 秒扫描 data/ipc/*/messages/              │
│         ↓                                                    │
│ 7. 发现 family-chat/messages/1708688400000-a3f2.json         │
│    来源群组 = 目录名 "family-chat"                             │
│         ↓                                                    │
│ 8. 读取 JSON，解析 chatJid 和 text                           │
│         ↓                                                    │
│ 9. 授权检查:                                                  │
│    sourceGroup="family-chat", targetJid 指向的群组也是          │
│    "family-chat" → ✓ 允许（给自己群组发消息）                   │
│         ↓                                                    │
│ 10. channel.sendMessage(chatJid, text)                       │
│     → WhatsApp 发送 "Andy: 天气晴朗，12°C"                   │
│         ↓                                                    │
│ 11. fs.unlinkSync(filePath)  ← 删除已处理的 IPC 文件          │
└─────────────────────────────────────────────────────────────┘
```

### 通道 2：容器 → 宿主机（任务操作）

```
场景: Agent 调用 schedule_task 工具

容器内 MCP Server:
  writeIpcFile('/workspace/ipc/tasks/', {
    type: 'schedule_task',        ← 操作类型
    prompt: '搜索AI新闻...',      ← 任务提示词
    schedule_type: 'cron',
    schedule_value: '0 9 * * 1',  ← 每周一早上9点
    context_mode: 'isolated',
    targetJid: '120363...@g.us',
    createdBy: 'family-chat',
  })

宿主机 IPC Watcher:
  1. 扫描 family-chat/tasks/ 目录
  2. 解析 JSON
  3. processTaskIpc(data, 'family-chat', false, deps)
     ↓
  4. switch(data.type) → 'schedule_task'
  5. 授权检查: 非 main + targetFolder === sourceGroup → ✓
  6. 计算 next_run = CronParser('0 9 * * 1').next()
  7. createTask({ id: 'task-xxx', ... })  → 写入 SQLite
  8. 删除 IPC 文件
```

**支持的任务操作类型：**

| type | 作用 | 授权规则 |
|------|------|---------|
| `schedule_task` | 创建定时任务 | 非 main 只能为自己创建 |
| `pause_task` | 暂停任务 | 非 main 只能操作自己群组的 |
| `resume_task` | 恢复任务 | 同上 |
| `cancel_task` | 取消任务 | 同上 |
| `register_group` | 注册新群组 | **仅 main** |
| `refresh_groups` | 刷新群组列表 | **仅 main** |

### 通道 3：宿主机 → 容器（消息注入）

```
场景: 容器正在运行，用户又发了一条新消息

┌─────────────────────────────────────────────────────────────┐
│ 宿主机 消息循环 (index.ts)                                    │
│                                                             │
│ 1. 检测到新消息（含触发词）                                     │
│ 2. 检查该群组是否有活跃容器: queue.sendMessage(chatJid, text)   │
│         ↓  (GroupQueue.sendMessage)                          │
│ 3. state.active === true 且 !isTaskContainer → 有活跃容器     │
│ 4. state.idleWaiting = false  ← 标记容器即将收到工作            │
│ 5. 原子写入:                                                  │
│    data/ipc/family-chat/input/1708688500000-d9e3.json        │
│    内容: { type: 'message', text: '<messages>...' }          │
│ 6. 返回 true（消息已送达）                                     │
└─────────────────────────────────────────────────────────────┘
         ↓  (文件在共享挂载卷上)
┌─────────────────────────────────────────────────────────────┐
│ 容器内 Agent Runner (index.ts)                               │
│                                                             │
│ 两种注入路径:                                                  │
│                                                             │
│ 路径 A — query() 正在运行:                                    │
│   pollIpcDuringQuery() 每 500ms 扫描 /workspace/ipc/input/   │
│   → 读取 JSON → stream.push(text)                           │
│   → Claude 在当前对话轮次中处理新消息                            │
│                                                             │
│ 路径 B — query() 已完成，等待 IPC:                             │
│   waitForIpcMessage() 每 500ms 扫描                          │
│   → 读取 JSON → 作为新 prompt 启动新一轮 query()               │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**路径 A vs 路径 B 的关键区别：**

| 方面 | 路径 A（查询期间注入） | 路径 B（查询间注入） |
|------|---------------------|---------------------|
| 时机 | Claude 正在思考/使用工具 | Claude 已完成，容器空闲 |
| 实现 | `stream.push(text)` | 新的 `runQuery(text, ...)` |
| 效果 | 消息作为新用户消息加入当前对话流 | 在同一会话中开始新一轮对话 |
| 延迟 | 极低（500ms 轮询间隔） | 低（500ms + query 启动时间） |
| 上下文 | 完全连续（同一次 API 调用链） | 通过 `resumeSessionAt` 恢复 |

### 通道 4：宿主机 → 容器（关闭信号）

```
场景: 30 分钟无活动，需要关闭容器

宿主机:
  1. IDLE_TIMEOUT (30分钟) 触发
  2. queue.closeStdin(chatJid)
  3. 创建哨兵文件: data/ipc/family-chat/input/_close  (空文件)

容器内:
  pollIpcDuringQuery() 或 waitForIpcMessage() 检测到 _close 文件
  → 删除哨兵文件
  → stream.end() 或返回 null
  → 查询循环退出
  → 容器进程结束
  → --rm 标志自动清理容器
```

## IPC 的三个安全保障

### 1. 目录即身份

```
容器 A (family-chat) 挂载的 IPC 目录:
  宿主机 data/ipc/family-chat/ → 容器内 /workspace/ipc/

容器 B (work-team) 挂载的 IPC 目录:
  宿主机 data/ipc/work-team/ → 容器内 /workspace/ipc/

两个容器看到的路径都是 /workspace/ipc/，但映射到不同的宿主机目录。
宿主机 IPC Watcher 通过遍历 data/ipc/ 下的子目录来确定身份。
容器无法伪造身份——它只能写入自己的 IPC 目录。
```

### 2. 原子写入

```typescript
// 容器端和宿主机端都使用相同的原子写入模式
const tempPath = `${filepath}.tmp`;
fs.writeFileSync(tempPath, JSON.stringify(data));  // 步骤1: 写临时文件
fs.renameSync(tempPath, filepath);                  // 步骤2: 原子重命名
```

如果不使用原子写入，IPC Watcher 可能读到一个写了一半的 JSON 文件，导致解析错误。`rename` 是操作系统保证的原子操作。

### 3. 错误隔离

```typescript
// ipc.ts - 处理失败的文件被移到 errors/ 目录
catch (err) {
    const errorDir = path.join(ipcBaseDir, 'errors');
    fs.mkdirSync(errorDir, { recursive: true });
    fs.renameSync(filePath, path.join(errorDir, `${sourceGroup}-${file}`));
}
```

一个损坏的 IPC 文件不会阻塞整个系统——它被移到 `errors/` 目录供调试，处理继续。

## 数据快照机制

除了双向通信，宿主机还通过**数据快照**向容器传递只读信息：

```typescript
// container-runner.ts — 每次启动容器前写入快照
writeTasksSnapshot(group.folder, isMain, tasks);     // → current_tasks.json
writeGroupsSnapshot(group.folder, isMain, groups);   // → available_groups.json
```

```
容器内 MCP Server 读取快照:
  list_tasks 工具 → 读取 /workspace/ipc/current_tasks.json
  register_group 工具 → 读取 /workspace/ipc/available_groups.json
```

快照是一种"最终一致性"模式——容器看到的可能不是最新数据，但这对于列出任务/群组这种操作来说完全可以接受。

## 时序图：一次完整的 IPC 交互

```
    用户          WhatsApp      宿主机          IPC文件系统        容器
     │               │            │                │              │
     │  @Andy 天气   │            │                │              │
     │──────────────>│            │                │              │
     │               │  消息存储   │                │              │
     │               │───────────>│                │              │
     │               │            │  轮询发现       │              │
     │               │            │  写任务快照     │              │
     │               │            │───────────────>│              │
     │               │            │  启动容器       │              │
     │               │            │───────────────────────────────>│
     │               │            │  stdin: JSON   │              │
     │               │            │───────────────────────────────>│
     │               │            │                │              │ 调用Claude
     │               │            │                │              │ SDK query()
     │               │            │                │              │
     │               │            │                │  send_message │
     │               │            │                │<─────────────│
     │               │            │  IPC扫描       │              │
     │               │            │<───────────────│              │
     │               │  发送消息   │                │              │
     │               │<───────────│                │              │
     │  Andy: 晴朗   │            │                │              │
     │<──────────────│            │                │              │
     │               │            │                │              │
     │               │            │                │  (等待IPC)    │
     │  再问一句      │            │                │              │
     │──────────────>│            │                │              │
     │               │───────────>│                │              │
     │               │            │  管道发送       │              │
     │               │            │───────────────>│              │
     │               │            │                │  (容器读取)   │
     │               │            │                │─────────────>│
     │               │            │                │              │ 继续query()
     │               │            │                │              │
     │               │            │  30分钟空闲     │              │
     │               │            │  写 _close     │              │
     │               │            │───────────────>│              │
     │               │            │                │  (检测到)     │
     │               │            │                │─────────────>│
     │               │            │                │              │ 退出
     │               │            │                │              │ ×
```

---

**下一篇：** [08 — NanoClaw 与 Claude Code 交互全流程](./08-nanoclaw-claude-code-interaction.md)
