# 06 — Claude Agent SDK 深度剖析

## SDK 的三层架构

Claude Agent SDK 不是一个简单的 API 调用封装。它有三层架构：

```
┌──────────────────────────────────────────────────────┐
│ 第一层: Agent Runner (我们的代码)                       │
│                                                      │
│  const result = query({                              │
│      prompt: messageStream,                          │
│      options: { cwd, tools, mcpServers, ... }        │
│  });                                                 │
│  for await (const msg of result) { /* 处理事件 */ }    │
│                                                      │
├──────────────────────────────────────────────────────┤
│ 第二层: SDK 运行时 (sdk.mjs)                          │
│                                                      │
│  - 启动 CLI 子进程                                     │
│  - stdin/stdout JSON-lines 通信                       │
│  - 异步生成器封装                                      │
│  - 控制 isSingleUserTurn 行为                          │
│                                                      │
├──────────────────────────────────────────────────────┤
│ 第三层: CLI 进程 (cli.js)                             │
│                                                      │
│  - EZ() 递归 Agent 循环                                │
│  - Claude API 调用                                    │
│  - 工具执行                                           │
│  - Agent Teams 子进程管理                              │
│  - 会话持久化                                          │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**关键认知：** SDK 运行时（我们在 `package.json` 中安装的 `@anthropic-ai/claude-agent-sdk`）并不直接调用 Claude API。它启动一个 CLI 子进程（`claude-code`），所有真正的逻辑都在这个子进程中运行。SDK 只是一个"传输层"。

## query() 函数：Agent 的入口

### 调用方式

NanoClaw 这样调用 `query()`：

```typescript
for await (const message of query({
    prompt: stream,        // AsyncIterable<SDKUserMessage> — 不是字符串！
    options: {
        cwd: '/workspace/group',
        resume: sessionId,
        allowedTools: ['Bash', 'Read', 'Write', 'Edit', 'WebSearch', ...],
        permissionMode: 'bypassPermissions',
        allowDangerouslySkipPermissions: true,
        settingSources: ['project', 'user'],
        mcpServers: { nanoclaw: { command: 'node', args: [...] } },
        hooks: {
            PreCompact: [{ hooks: [createPreCompactHook()] }],
            PreToolUse: [{ matcher: 'Bash', hooks: [createSanitizeBashHook()] }],
        },
        env: sdkEnv,
    }
})) {
    // 处理流式消息
}
```

### 关键参数详解

**`prompt: AsyncIterable`（而非字符串）**

这是 NanoClaw 最关键的设计决策之一。当 prompt 是字符串时：
- SDK 设置 `isSingleUserTurn = true`
- 第一个 result 消息后自动关闭 CLI 的 stdin
- 如果有 Agent Teams 的子 Agent 在运行，它们会被强制终止

当 prompt 是 AsyncIterable 时：
- `isSingleUserTurn = false`
- stdin 保持打开，子 Agent 可以继续运行
- 我们控制何时结束迭代

NanoClaw 用 `MessageStream` 类实现了这个 AsyncIterable：

```typescript
class MessageStream {
    private queue: SDKUserMessage[] = [];
    private waiting: (() => void) | null = null;
    private done = false;

    push(text: string): void {
        this.queue.push({
            type: 'user',
            message: { role: 'user', content: text },
            parent_tool_use_id: null,
            session_id: '',
        });
        this.waiting?.(); // 唤醒等待中的消费者
    }

    end(): void {
        this.done = true;
        this.waiting?.();
    }

    async *[Symbol.asyncIterator](): AsyncGenerator<SDKUserMessage> {
        while (true) {
            while (this.queue.length > 0) yield this.queue.shift()!;
            if (this.done) return;
            await new Promise<void>(r => { this.waiting = r; }); // 等待新消息
            this.waiting = null;
        }
    }
}
```

**`permissionMode: 'bypassPermissions'`**

在正常的 Claude Code 使用中，某些操作（如写文件、执行危险命令）需要用户确认。在容器化的 Agent 中，没有人类可以确认——也不需要，因为容器本身就是安全边界。`bypassPermissions` 让 Agent 自由使用所有工具。

**`settingSources: ['project', 'user']`**

告诉 SDK 加载哪些设置文件：
- `'project'` → 加载工作目录中的 `.claude/settings.json` 和 `CLAUDE.md`
- `'user'` → 加载 `~/.claude/settings.json`（容器内是 `/home/node/.claude/`）

这就是 NanoClaw 的记忆系统的基础——Agent 启动时自动读取 CLAUDE.md。

**`resume: sessionId`**

恢复之前的会话。Claude SDK 内部通过会话 ID 找到之前的 JSONL 转录文件，将对话历史重放到 Claude 的上下文中。这让 Agent 能"记住"之前的对话。

## EZ()：Agent 的核心循环

在 CLI 进程内部，所有的智能行为都由一个名为 `EZ()` 的递归异步生成器驱动：

```
EZ({ messages, systemPrompt, canUseTool, maxTurns, turnCount=1, ... })

每次调用 = 一次 Claude API 请求

┌─────────────────────────────────────────────────┐
│ EZ() 的单次迭代：                                 │
│                                                 │
│  1. 准备消息（裁剪上下文、压缩如需）                  │
│  2. 调用 Claude API（流式响应）                    │
│  3. 解析响应，提取 tool_use 块                      │
│  4. 判断：                                        │
│     ├── 有 tool_use → 执行工具 → turnCount++ → 递归 EZ()  │
│     └── 无 tool_use → 任务完成 → 返回结果             │
│                                                 │
└─────────────────────────────────────────────────┘
```

### Agent 何时停止？

| 条件 | Agent 行为 | 结果类型 |
|------|-----------|---------|
| 响应无 tool_use 块 | 停止（Claude 认为任务完成） | `success` |
| 响应有 tool_use 块 | 执行工具，继续 | 继续循环 |
| 超过 maxTurns | 强制停止 | `error_max_turns` |
| 超过 maxBudgetUsd | 强制停止 | `error_max_budget_usd` |
| abort 信号 | 中断 | 取决于上下文 |
| max_tokens 达到 | 最多重试 3 次 | 继续或停止 |

**核心洞察：** Agent 的"自主性"体现在这里——SDK 不决定何时停止，Claude 模型自己决定。当 Claude 认为任务完成了，它会生成一个不包含 tool_use 块的响应，循环自然终止。

### 工具执行流程

```
Claude API 返回:
{
    "content": [
        { "type": "text", "text": "让我搜索一下天气" },
        { "type": "tool_use", "id": "toolu_123", "name": "WebSearch", "input": { "query": "上海今天天气" } }
    ]
}

SDK 处理:
1. 识别 tool_use 块
2. 执行 WebSearch 工具
3. 构建工具结果消息:
{
    "role": "user",
    "content": [
        { "type": "tool_result", "tool_use_id": "toolu_123", "content": "上海今天晴，12°C..." }
    ]
}
4. 将结果追加到消息历史
5. 递归调用 EZ()（下一轮 API 请求）
```

## 消息类型全景

`query()` 返回的 AsyncGenerator 产出 16 种消息类型。NanoClaw 主要关注其中几种：

### 重要消息类型

```typescript
for await (const message of query(...)) {
    switch (message.type) {
        case 'system':
            if (message.subtype === 'init') {
                // 会话初始化，获取 session_id
                newSessionId = message.session_id;
            }
            if (message.subtype === 'task_notification') {
                // Agent Teams 子 Agent 完成/失败
                // task_id, status: 'completed'|'failed'|'stopped'
            }
            break;
            
        case 'assistant':
            // Claude 的响应（文本 + 工具调用）
            lastAssistantUuid = message.uuid; // 用于 resumeSessionAt
            break;
            
        case 'result':
            // 一轮交互完成
            if (message.subtype === 'success') {
                writeOutput({ status: 'success', result: message.result });
            }
            // 可能有多个 result（Agent Teams 场景）
            break;
    }
}
```

### 消息流的典型序列

**简单查询：**
```
system/init       → 会话初始化
assistant         → Claude 思考并回复
result/success    → "今天天气晴朗，12°C"
[generator done]
```

**使用工具的查询：**
```
system/init       → 会话初始化
assistant         → Claude: "让我搜索一下" + tool_use(WebSearch)
assistant         → Claude: "根据搜索结果..." (无 tool_use)
result/success    → "上海今天天气晴朗..."
[generator done]
```

**Agent Teams 场景：**
```
system/init            → 会话初始化
assistant              → Leader: 创建团队
assistant              → Leader: 分配任务给子 Agent
result #1              → Leader: "我已经派出团队调研..."
task_notification      → Researcher A 完成
assistant              → Leader: 处理 A 的结果
task_notification      → Researcher B 完成
assistant              → Leader: 汇总所有结果
result #2              → Leader: "综合调研结果如下..."
[generator done]       → 所有子 Agent 关闭
```

## 会话管理：对话的连续性

### 会话 ID 与恢复

```
第一次对话:
  query({ prompt: "你好" })
  → system/init: session_id = "abc123"
  → result: "你好！"
  
  保存: sessions["family-chat"] = "abc123"

第二次对话:
  query({ prompt: "记得我说的吗？", resume: "abc123" })
  → SDK 加载 abc123 的 JSONL 转录
  → 重放之前的对话历史到 Claude 上下文
  → Claude 可以引用之前的对话
  → result: "当然，你之前说了'你好'..."
```

### resumeSessionAt

NanoClaw 使用 `resumeSessionAt` 来精确控制恢复点：

```typescript
let resumeAt: string | undefined;

const queryResult = await runQuery(prompt, sessionId, ...);
if (queryResult.lastAssistantUuid) {
    resumeAt = queryResult.lastAssistantUuid; // 记住最后一条助手消息的 UUID
}

// 下次 query:
await runQuery(newPrompt, sessionId, ..., resumeAt);
// SDK 从 resumeAt 指定的消息之后恢复，避免重放冗余历史
```

### 会话压缩 (Compaction)

当对话历史太长（接近上下文窗口限制）时，SDK 自动压缩：

```
压缩前:
[消息1] [消息2] [消息3] ... [消息100] [消息101]

压缩后:
[摘要: 用户讨论了天气、健身计划、旅行安排...] [消息100] [消息101]
```

NanoClaw 通过 PreCompact Hook 在压缩前归档完整对话到 Markdown 文件，确保信息不会丢失。

## 容器内的查询循环

NanoClaw 的 Agent Runner 不只执行一次 `query()`，而是运行一个查询循环：

```
┌─────────────────────────────────────────────────────┐
│               Agent Runner 查询循环                    │
│                                                     │
│  1. 读取 stdin（初始输入 + secrets）                   │
│       ↓                                             │
│  2. 调用 runQuery(prompt, sessionId)                │
│       ↓                                             │
│  3. 输出结果（OUTPUT_START/END 标记对）                │
│       ↓                                             │
│  4. 发出会话更新标记（result: null, newSessionId）     │
│       ↓                                             │
│  5. waitForIpcMessage()                             │
│     ├── 收到 _close 哨兵 → 退出循环                   │
│     └── 收到新消息      → 更新 prompt → 回到 2        │
│                                                     │
│  特殊: 在 query() 运行期间也轮询 IPC                   │
│  └── 新消息通过 MessageStream.push() 注入活跃查询       │
│  └── _close 哨兵触发 MessageStream.end()               │
│                                                     │
└─────────────────────────────────────────────────────┘
```

**两种消息注入方式：**

1. **查询期间注入**：query() 正在运行时收到新消息 → 通过 `stream.push()` 注入 → Claude 在同一轮对话中处理
2. **查询间注入**：query() 完成后收到新消息 → 作为新的 prompt 开始新一轮 query()

这种设计让容器可以长期保持活跃，处理多轮对话，而不需要为每条消息启动新容器。

## 总结：从 SDK 看 Agent 工程的核心挑战

通过深入分析 Claude Agent SDK，我们可以看到 Agent 工程的几个核心挑战：

### 1. 循环控制

Agent 的核心是一个"推理→行动→观察"循环。关键问题是何时停止——NanoClaw 依赖 Claude 模型自身的判断（响应中是否包含 tool_use 块），同时设置 timeout 和 budget 作为安全边界。

### 2. 上下文管理

对话越长，上下文窗口压力越大。SDK 的自动压缩 + NanoClaw 的 PreCompact 归档 + resumeSessionAt 精确恢复，共同解决了这个问题。

### 3. 并发与生命周期

Agent Teams 的子 Agent 异步运行，Leader Agent 需要等待它们完成。`isSingleUserTurn` 的问题是一个经典的生命周期管理 Bug——stdin 过早关闭导致子进程被杀。NanoClaw 通过 AsyncIterable 解决了这个问题。

### 4. 可靠性

容器可能超时、API 可能失败、网络可能断开。NanoClaw 的应对：
- 超时后优雅终止容器
- 双游标机制支持崩溃恢复
- 指数退避重试（GroupQueue）
- 输出已发送后不回滚游标（避免重复发送）

### 5. 安全性

Agent 有执行代码的能力，安全至关重要。NanoClaw 的层层防御：
- 容器隔离（操作系统级）
- 挂载安全（外部白名单）
- 凭证隔离（stdin 传递 + Bash Hook 清除）
- IPC 授权（目录路径验证）

---

**恭喜！** 你已经完成了所有学习文档。回到 [目录](./00-INDEX.md) 复习关键概念。
