# 11 — 定时任务与轮询调度系统

## 概述

NanoClaw 有两套独立的轮询循环在宿主机上持续运行：

| 循环 | 文件 | 轮询间隔 | 职责 |
|------|------|---------|------|
| **消息轮询** | `index.ts → startMessageLoop()` | 2 秒 | 从 SQLite 读新消息，派发给容器 |
| **任务调度** | `task-scheduler.ts → startSchedulerLoop()` | 60 秒 | 检查到期任务，入队执行 |

此外还有一个辅助循环：

| 循环 | 文件 | 轮询间隔 | 职责 |
|------|------|---------|------|
| **IPC 轮询** | `ipc.ts → startIpcWatcher()` | 1 秒 | 扫描容器写的 IPC 文件 |

三个循环**同时运行**，通过 `GroupQueue` 协调并发。

## 系统启动时序

```typescript
// index.ts → main()

async function main(): Promise<void> {
  ensureContainerSystemRunning();  // 确保 Docker 可用
  initDatabase();                   // 初始化 SQLite 表
  loadState();                      // 加载游标、会话、群组注册

  // 连接通信渠道
  await whatsapp.connect();

  // 启动三个并行子系统:
  startSchedulerLoop(deps);         // ① 任务调度循环 (60s)
  startIpcWatcher(deps);            // ② IPC 轮询循环 (1s)
  recoverPendingMessages();         // ③ 崩溃恢复: 检查积压消息
  startMessageLoop();               // ④ 消息轮询循环 (2s)
}
```

```
        main()
          │
          ├─ ensureContainerSystemRunning()
          ├─ initDatabase()
          ├─ loadState()
          ├─ whatsapp.connect()
          │
          ├─────────────────────────────────────────────────┐
          │                                                 │
          ▼                    ▼                    ▼       ▼
    schedulerLoop()     ipcWatcher()       recover()   messageLoop()
      每 60s               每 1s             一次性       每 2s
      检查到期任务          扫描 IPC 文件     恢复积压     检查新消息
          │                   │                            │
          │                   │                            │
          ▼                   ▼                            ▼
       GroupQueue.enqueueTask()                GroupQueue.enqueueMessageCheck()
                        \                       /
                         ▼                     ▼
                        GroupQueue (并发协调器)
                              │
                              ▼
                     runContainerAgent()
```

## 第一部分：消息轮询循环

### 轮询流程

```typescript
// index.ts → startMessageLoop()

while (true) {
  // 1. 查询所有注册群组的新消息
  const { messages, newTimestamp } = getNewMessages(jids, lastTimestamp, ASSISTANT_NAME);
  //    ↑ SQL: SELECT ... WHERE timestamp > lastTimestamp AND chat_jid IN (...)

  if (messages.length > 0) {
    // 2. 推进"已看到"游标 (防止重复扫描)
    lastTimestamp = newTimestamp;
    saveState();

    // 3. 按群组分组
    const messagesByGroup = new Map<string, NewMessage[]>();
    for (const msg of messages) {
      messagesByGroup.get(msg.chat_jid)?.push(msg) || messagesByGroup.set(msg.chat_jid, [msg]);
    }

    // 4. 逐群组处理
    for (const [chatJid, groupMessages] of messagesByGroup) {
      // 4a. 非 main 群组需要检查触发词
      if (needsTrigger && !hasTrigger) continue;

      // 4b. 尝试管道到活跃容器
      if (queue.sendMessage(chatJid, formatted)) {
        // ✅ 消息已注入活跃容器
      } else {
        // ❌ 无活跃容器 → 入队，等待容器槽位
        queue.enqueueMessageCheck(chatJid);
      }
    }
  }

  // 5. 等待 2 秒后继续
  await sleep(POLL_INTERVAL);  // 2000ms
}
```

### 双游标机制

NanoClaw 使用两个独立的时间戳游标：

```
lastTimestamp:       消息轮询的"已看到"标记
lastAgentTimestamp:  每个群组的"已处理"标记 (per-group)

为什么需要两个？

lastTimestamp 跟踪的是:
  "我已经扫描到这个时间点的所有消息了"
  → 下次轮询从这之后开始

lastAgentTimestamp[chatJid] 跟踪的是:
  "这个群组的 Agent 已经处理到这个时间点了"
  → 下次启动容器时，从这之后的消息作为 prompt

场景:
  t=1 消息 A (main)        → lastTimestamp = t1
  t=2 消息 B (family-chat)  → lastTimestamp = t2
  t=3 消息 C (family-chat)  → lastTimestamp = t3

  Agent 处理 family-chat:
    读取 B + C → lastAgentTimestamp['family-chat'] = t3
  Agent 处理 main:
    读取 A → lastAgentTimestamp['main'] = t1

  两个群组的处理进度完全独立
```

### 上下文积累

非 main 群组的消息不是每条都触发 Agent——只有包含触发词的消息才会激活。但积累的消息不会丢失：

```typescript
// 触发时，读取 lastAgentTimestamp 之后的所有消息 (不只是触发消息)
const allPending = getMessagesSince(chatJid, lastAgentTimestamp[chatJid] || '', ASSISTANT_NAME);
```

```
t=1  "今天天气真好"          → 存入 SQLite，不触发
t=2  "周末去哪玩？"          → 存入 SQLite，不触发
t=3  "@Andy 帮我规划周末行程"  → 触发！

Agent 收到的 prompt 包含所有 3 条消息:
  <messages>
    <message from="小明" time="t1">今天天气真好</message>
    <message from="小红" time="t2">周末去哪玩？</message>
    <message from="小明" time="t3">@Andy 帮我规划周末行程</message>
  </messages>

Agent 看到完整上下文，知道大家在讨论什么
```

## 第二部分：任务调度系统

### 任务的生命周期

```
                创建                    到期检查                  执行                    后续
                ────                    ──────                  ────                    ────
                                                                                        
  Agent 调用      IPC        SQLite      调度器        GroupQueue      容器运行        更新 next_run
  schedule_task → 文件 ──→  写入任务  ←── 轮询发现 ───→ enqueueTask ──→ 启动容器 ──→   计算下次时间
                                                                        │
                                                                        ▼
                                                                  结果发送给用户
                                                                        │
                                                                        ▼
                                                                  logTaskRun()
                                                                  记录执行日志
```

### 任务数据模型

```sql
CREATE TABLE scheduled_tasks (
  id TEXT PRIMARY KEY,           -- 'task-1708688400000-a3f2'
  group_folder TEXT NOT NULL,    -- 'family-chat'
  chat_jid TEXT NOT NULL,        -- '120363...@g.us'
  prompt TEXT NOT NULL,          -- 'Search for AI news and summarize'
  schedule_type TEXT NOT NULL,   -- 'cron' | 'interval' | 'once'
  schedule_value TEXT NOT NULL,  -- '0 9 * * *' | '3600000' | '2026-03-01T15:00:00'
  context_mode TEXT DEFAULT 'isolated',  -- 'group' | 'isolated'
  next_run TEXT,                 -- '2026-02-24T09:00:00.000Z' (下次执行时间)
  last_run TEXT,                 -- 上次执行时间
  last_result TEXT,              -- 上次执行结果摘要 (前 200 字符)
  status TEXT DEFAULT 'active',  -- 'active' | 'paused' | 'completed'
  created_at TEXT NOT NULL
);

CREATE TABLE task_run_logs (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  task_id TEXT NOT NULL,
  run_at TEXT NOT NULL,
  duration_ms INTEGER,
  status TEXT NOT NULL,          -- 'success' | 'error'
  result TEXT,
  error TEXT,
  FOREIGN KEY (task_id) REFERENCES scheduled_tasks(id)
);
```

### 三种调度类型

```
┌──────────────────────────────────────────────────────────────────┐
│ cron: 基于时间表达式                                               │
│                                                                  │
│ schedule_value: "0 9 * * 1"  (每周一早上 9 点)                     │
│                                                                  │
│ 创建时: next_run = CronParser.parse("0 9 * * 1").next()          │
│   → "2026-02-24T01:00:00.000Z" (UTC，实际是北京时间 09:00)         │
│                                                                  │
│ 执行后: next_run = CronParser.parse("0 9 * * 1").next()          │
│   → "2026-03-03T01:00:00.000Z" (下一个周一)                       │
│                                                                  │
│ 适用: 每天早报、每周总结、定时提醒                                    │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│ interval: 固定间隔                                                │
│                                                                  │
│ schedule_value: "3600000"  (每 1 小时 = 3600000ms)                │
│                                                                  │
│ 创建时: next_run = Date.now() + 3600000                          │
│ 执行后: next_run = Date.now() + 3600000                          │
│                                                                  │
│ 注意: 间隔是从执行完成时刻开始计算，不是从上次 next_run 开始           │
│ 如果任务执行了 5 分钟，实际间隔是 65 分钟                              │
│                                                                  │
│ 适用: 定期检查、持续监控                                             │
└──────────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────────┐
│ once: 一次性执行                                                   │
│                                                                  │
│ schedule_value: "2026-03-01T15:00:00"  (本地时间，无 Z 后缀)       │
│                                                                  │
│ 创建时: next_run = new Date("2026-03-01T15:00:00").toISOString() │
│ 执行后: next_run = null → status 自动变为 'completed'             │
│                                                                  │
│ 适用: 延迟提醒、定时发送、一次性检查                                   │
└──────────────────────────────────────────────────────────────────┘
```

### 调度循环详解

```typescript
// task-scheduler.ts → startSchedulerLoop()

const loop = async () => {
  // 1. 查询到期任务
  const dueTasks = getDueTasks();
  //    SQL: SELECT * FROM scheduled_tasks
  //         WHERE status = 'active' AND next_run IS NOT NULL AND next_run <= now()
  //         ORDER BY next_run

  for (const task of dueTasks) {
    // 2. 双重检查状态 (可能在上次循环和现在之间被暂停/取消)
    const currentTask = getTaskById(task.id);
    if (!currentTask || currentTask.status !== 'active') continue;

    // 3. 入队到 GroupQueue (遵守并发限制)
    deps.queue.enqueueTask(
      currentTask.chat_jid,
      currentTask.id,
      () => runTask(currentTask, deps),
    );
  }

  // 4. 60 秒后再次检查
  setTimeout(loop, SCHEDULER_POLL_INTERVAL);  // 60000ms
};
```

### 任务执行详解

```typescript
// task-scheduler.ts → runTask()

async function runTask(task: ScheduledTask, deps: SchedulerDependencies): Promise<void> {
  // 1. 确定上下文模式
  const sessionId = task.context_mode === 'group'
    ? sessions[task.group_folder]  // 使用群组的会话 (有对话历史)
    : undefined;                    // 全新会话 (无历史)

  // 2. 启动容器
  const output = await runContainerAgent(group, {
    prompt: task.prompt,
    sessionId,
    isScheduledTask: true,   // ← 标记为定时任务
    ...
  });

  // 3. 处理输出 (任务容器有特殊的关闭逻辑)
  // 任务产生结果后 10 秒自动关闭容器
  // (普通消息容器等待 30 分钟空闲超时)
  const TASK_CLOSE_DELAY_MS = 10000;

  // 4. 记录执行日志
  logTaskRun({
    task_id: task.id,
    run_at: new Date().toISOString(),
    duration_ms: durationMs,
    status: error ? 'error' : 'success',
    result,
    error,
  });

  // 5. 计算下次执行时间
  if (task.schedule_type === 'cron') {
    nextRun = CronParser.parse(task.schedule_value, { tz: TIMEZONE }).next();
  } else if (task.schedule_type === 'interval') {
    nextRun = new Date(Date.now() + parseInt(task.schedule_value));
  }
  // once → nextRun = null → status = 'completed'

  // 6. 更新任务记录
  updateTaskAfterRun(task.id, nextRun, resultSummary);
  //    SQL: UPDATE scheduled_tasks
  //         SET next_run=?, last_run=?, last_result=?,
  //             status = CASE WHEN next_run IS NULL THEN 'completed' ELSE status END
}
```

### 任务容器 vs 消息容器

| 特征 | 消息容器 | 任务容器 |
|------|---------|---------|
| 触发方式 | 用户消息 | 调度器到期 |
| prompt 来源 | XML 格式化的群组消息 | 任务的 prompt 字段 |
| prompt 前缀 | 无 | `[SCHEDULED TASK - ...]` |
| session | 群组会话 (总是) | 取决于 context_mode |
| 空闲超时 | 30 分钟 | 10 秒 (结果产生后) |
| 接受管道消息 | 是 | **否** (isTaskContainer=true) |
| 输出行为 | 直接发给用户 | 通过 `sendMessage` 或 `send_message` MCP 工具 |

### 容器内如何识别定时任务

```typescript
// container/agent-runner/src/index.ts → main()

let prompt = containerInput.prompt;
if (containerInput.isScheduledTask) {
  prompt = `[SCHEDULED TASK - The following message was sent automatically and is not coming directly from the user or group.]\n\n${prompt}`;
}
```

这个前缀告诉 Claude：
- 这不是用户直接发的消息
- 不需要"对话式"回复
- 应该专注于执行任务指令

## 第三部分：任务创建流程（从用户到 SQLite）

```
用户: "@Andy 每天早上 9 点给我发天气预报"

步骤 1 — 消息处理
  WhatsApp → SQLite → 消息循环 → 容器启动 → Claude 理解意图

步骤 2 — Claude 决定创建任务
  Claude 调用 mcp__nanoclaw__schedule_task 工具:
  {
    prompt: "查询天气并发送预报",
    schedule_type: "cron",
    schedule_value: "0 9 * * *",
    context_mode: "isolated"
  }

步骤 3 — MCP Server 验证并写 IPC
  容器内 ipc-mcp-stdio.ts:
    a. 验证 cron 表达式合法性
    b. 写 IPC 文件: /workspace/ipc/tasks/1708688400000-a3f2.json
    c. 返回 Claude: "Task scheduled"

步骤 4 — 宿主机 IPC Watcher 处理
  ipc.ts → processTaskIpc():
    a. 读取 JSON
    b. 授权检查
    c. 计算 next_run = CronParser("0 9 * * *").next()
    d. createTask() → INSERT INTO scheduled_tasks

步骤 5 — 调度循环发现到期任务
  task-scheduler.ts → getDueTasks():
    SELECT * FROM scheduled_tasks WHERE next_run <= now()
    → 找到该任务

步骤 6 — 执行
  queue.enqueueTask() → runTask() → runContainerAgent()
    prompt: "[SCHEDULED TASK] 查询天气并发送预报"
    → Claude 查询天气
    → Claude 调用 send_message 发送结果

步骤 7 — 更新
  next_run = 明天 09:00
  last_run = 今天 09:03
  last_result = "今天晴转多云，最高温度 15°C..."
```

## 第四部分：任务管理工具

容器内的 Agent 可以通过 MCP 工具管理任务：

```
┌──────────────────────────────────────────────────────────────┐
│ list_tasks                                                    │
│ 读取 /workspace/ipc/current_tasks.json (宿主机启动容器前写入)  │
│ main 群组: 看到所有任务                                        │
│ 其他群组: 只看到自己群组的任务                                   │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ schedule_task                                                 │
│ 写 IPC 文件到 /workspace/ipc/tasks/                           │
│ 容器端验证: cron 表达式、interval 正整数、once 时间戳格式        │
│ 宿主机端验证: 授权检查 + 重复验证 + 写入 SQLite                  │
└──────────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────────┐
│ pause_task / resume_task / cancel_task                        │
│ 写 IPC 文件，宿主机端执行:                                      │
│   pause  → UPDATE status = 'paused'  (调度器不会执行)          │
│   resume → UPDATE status = 'active'  (恢复调度)               │
│   cancel → DELETE FROM scheduled_tasks + task_run_logs        │
└──────────────────────────────────────────────────────────────┘
```

## 第五部分：轮询 vs 事件驱动的设计选择

NanoClaw 全面使用**轮询**而非事件驱动。这是有意的设计决策：

### 为什么不用事件驱动？

| 方面 | 事件驱动 | 轮询 |
|------|---------|------|
| 复杂度 | WebSocket/长连接/回调链 | setTimeout 循环 |
| 故障恢复 | 需要重连逻辑、心跳、断线缓冲 | 下一次轮询自动恢复 |
| 调试 | 难以重现时序问题 | 状态在 SQLite 中，随时可查 |
| 一致性 | 需要考虑并发修改 | 单线程轮询天然串行 |
| 延迟 | 毫秒级 | 秒级 (对聊天场景足够) |

### 三个循环的性能影响

```
消息轮询 (2s): 
  一次 SQLite SELECT → ~0.1ms
  2 秒轮询 → CPU 开销可忽略

任务调度 (60s):
  一次 SQLite SELECT → ~0.1ms  
  60 秒轮询 → 几乎无开销

IPC 扫描 (1s):
  fs.readdirSync → ~1ms (目录通常为空或只有几个文件)
  1 秒轮询 → 很小的 I/O 开销
```

总的 CPU 开销远低于 0.1%。对于一个聊天 Agent 系统来说，2 秒的消息检测延迟和 60 秒的任务检测延迟完全可以接受。

## 完整时序图：定时任务的一生

```
    用户          容器A(消息)      宿主机         SQLite          容器B(任务)
     │               │              │              │               │
     │ @Andy 每天    │              │              │               │
     │ 9点发天气     │              │              │               │
     │──────────────>│              │              │               │
     │               │              │              │               │
     │               │ schedule_task│              │               │
     │               │  (IPC文件)   │              │               │
     │               │─────────────>│              │               │
     │               │              │              │               │
     │               │              │ createTask() │               │
     │               │              │─────────────>│               │
     │               │              │              │               │
     │   "已安排!"   │              │              │               │
     │<──────────────│              │              │               │
     │               │              │              │               │
     │               × (容器退出)    │              │               │
     │                              │              │               │
     ·  ·  ·  ·  (等到第二天 09:00)  ·  ·  ·  ·   │               │
     │                              │              │               │
     │                              │ getDueTasks()│               │
     │                              │<─────────────│               │
     │                              │              │               │
     │                              │ enqueueTask()│               │
     │                              │──────────────────────────────>│
     │                              │              │               │
     │                              │              │        Claude 查询天气
     │                              │              │               │
     │                              │  send_message│               │
     │                              │  (IPC文件)   │               │
     │                              │<─────────────────────────────│
     │                              │              │               │
     │  "今天晴，15°C"              │              │               │
     │<─────────────────────────────│              │               │
     │                              │              │               │
     │                              │ updateTaskAfterRun()         │
     │                              │─────────────>│               │
     │                              │ next_run=明天 │               │
     │                              │              │               │
     │                              │              │  10s后自动关闭  │
     │                              │──────────────────────────────>│
     │                              │              │               × 
```
