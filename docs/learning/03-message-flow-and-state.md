# 03 — 消息流转与状态管理

## 一条消息的完整旅程

让我们跟踪一条消息，从用户在 WhatsApp 中输入到收到 Agent 回复的完整生命周期。

### 场景

用户 John 在"家庭群"中发送：`@Andy 今天天气怎么样？`

### Step 1: WhatsApp 接收 → SQLite 存储

```
John 的手机 ──WhatsApp协议──→ baileys 库 (whatsapp.ts)
```

`WhatsAppChannel` 的 `messages.upsert` 事件处理器被触发：

```typescript
// whatsapp.ts - 消息接收
this.sock.ev.on('messages.upsert', async ({ messages }) => {
    for (const msg of messages) {
        const chatJid = await this.translateJid(rawJid);
        const content = msg.message?.conversation || msg.message?.extendedTextMessage?.text || '';
        
        // 只有已注册群组的消息才会存储内容
        if (groups[chatJid]) {
            this.opts.onMessage(chatJid, {
                id: msg.key.id,
                chat_jid: chatJid,
                sender: msg.key.participant,
                sender_name: msg.pushName || 'Unknown',
                content: '@Andy 今天天气怎么样？',
                timestamp: '2026-02-23T10:30:00.000Z',
                is_from_me: false,
                is_bot_message: false,  // 不是机器人发的
            });
        }
        
        // 所有聊天的元数据都会存储（用于群组发现）
        this.opts.onChatMetadata(chatJid, timestamp, undefined, 'whatsapp', true);
    }
});
```

`onMessage` 回调将消息写入 SQLite：

```sql
INSERT INTO messages (id, chat_jid, sender, sender_name, content, timestamp, is_from_me, is_bot_message)
VALUES ('msg-123', '120363...@g.us', '8613900001234@s.whatsapp.net', 'John', '@Andy 今天天气怎么样？', '2026-02-23T10:30:00.000Z', 0, 0)
```

### Step 2: 消息循环检测新消息

`startMessageLoop()` 每 2 秒执行一次：

```typescript
// index.ts - 消息循环（简化）
while (true) {
    const { messages, newTimestamp } = getNewMessages(jids, lastTimestamp, ASSISTANT_NAME);
    
    if (messages.length > 0) {
        // 按群组分组
        for (const [chatJid, groupMessages] of messagesByGroup) {
            // 检查触发词
            const hasTrigger = groupMessages.some(m => TRIGGER_PATTERN.test(m.content.trim()));
            // '@Andy 今天天气怎么样？' → 匹配 /^@Andy\b/i → hasTrigger = true
            
            if (hasTrigger) {
                // 拉取自上次 Agent 交互以来的所有消息（包括之前非触发的上下文消息）
                const allPending = getMessagesSince(chatJid, lastAgentTimestamp[chatJid], 'Andy');
                
                // 尝试管道发送到活跃容器，或排队启动新容器
                if (queue.sendMessage(chatJid, formatted)) {
                    // 已有活跃容器，消息通过 IPC 传入
                } else {
                    queue.enqueueMessageCheck(chatJid); // 排队
                }
            }
        }
    }
    await sleep(2000); // 等待 2 秒
}
```

### Step 3: 对话追赶 (Conversation Catch-up)

假设在 John 的消息之前，群里还有两条未处理的消息：

```
[10:25] Alice: 有人下午去健身吗？
[10:28] Bob: 我去！
[10:30] John: @Andy 今天天气怎么样？
```

NanoClaw 会拉取自上次 Agent 交互以来的**所有**消息，格式化为 XML：

```xml
<messages>
<message sender="Alice" time="2026-02-23T10:25:00.000Z">有人下午去健身吗？</message>
<message sender="Bob" time="2026-02-23T10:28:00.000Z">我去！</message>
<message sender="John" time="2026-02-23T10:30:00.000Z">@Andy 今天天气怎么样？</message>
</messages>
```

**为什么需要追赶？** 这让 Agent 理解对话上下文。即使没被直接 @ 的消息，也包含了有用的信息。Agent 可以看到完整的对话流，给出更有上下文的回答。

### Step 4: GroupQueue 调度

```
queue.enqueueMessageCheck(chatJid)
    │
    ▼
GroupQueue 检查:
    1. 该群组有活跃容器吗？ → 没有
    2. 当前运行的容器数 < 5 吗？ → 是
    3. 直接启动！
    │
    ▼
runForGroup(chatJid, 'messages')
    │
    ▼
processGroupMessages(chatJid)  ← index.ts 中注册的回调
```

### Step 5: 启动容器

`container-runner.ts` 构建容器启动参数：

```bash
docker run -i --rm \
    --name nanoclaw-family-chat-1708688400000 \
    -e TZ=Asia/Shanghai \
    --user 501:20 -e HOME=/home/node \
    -v /path/to/groups/family-chat:/workspace/group \                     # 群组文件夹（读写）
    --mount "type=bind,source=/path/to/groups/global,target=/workspace/global,readonly" \  # 全局记忆（只读）
    -v /path/to/data/sessions/family-chat/.claude:/home/node/.claude \    # Claude会话
    -v /path/to/data/ipc/family-chat:/workspace/ipc \                    # IPC通信
    -v /path/to/data/sessions/family-chat/agent-runner-src:/app/src \    # Agent Runner源码
    nanoclaw-agent:latest
```

容器启动后，输入 JSON 通过 stdin 传入：

```json
{
    "prompt": "<messages>\n<message sender=\"Alice\" time=\"...\">有人下午去健身吗？</message>\n...\n</messages>",
    "sessionId": "session-abc123",
    "groupFolder": "family-chat",
    "chatJid": "120363...@g.us",
    "isMain": false,
    "assistantName": "Andy",
    "secrets": {
        "CLAUDE_CODE_OAUTH_TOKEN": "sk-ant-oat01-..."
    }
}
```

**注意：** secrets 通过 stdin 传入，从不写入磁盘或环境变量，容器进程读取后立即删除临时文件。

### Step 6: Agent Runner 处理

在容器内，`agent-runner/src/index.ts` 读取输入并启动 Claude Agent SDK：

```typescript
// 读取 stdin
const containerInput = JSON.parse(await readStdin());

// 调用 Claude Agent SDK
for await (const message of query({
    prompt: messageStream,  // AsyncIterable（不是字符串！避免 isSingleUserTurn 问题）
    options: {
        cwd: '/workspace/group',
        resume: sessionId,       // 恢复之前的会话
        allowedTools: ['Bash', 'Read', 'Write', 'Edit', 'WebSearch', 'WebFetch', ...],
        permissionMode: 'bypassPermissions',  // 跳过权限确认（在容器内安全）
        settingSources: ['project', 'user'],  // 加载 CLAUDE.md
        mcpServers: { nanoclaw: { command: 'node', args: [mcpServerPath] } },
    }
})) {
    if (message.type === 'result') {
        // 输出结果
        writeOutput({ status: 'success', result: message.result, newSessionId });
    }
}
```

### Step 7: Claude 的推理过程

Claude 收到消息后，内部推理可能如下：

```
输入: 群组对话，John 在问天气

思考: 用户问天气，我需要使用 WebSearch 工具查询当前天气

→ 工具调用: WebSearch("今天天气")
← 工具结果: "上海今天晴，最高温度12°C..."

思考: 我有了天气信息，可以回复了

→ 最终回复: "今天上海天气晴朗，最高温度12°C，适合户外活动！Alice和Bob如果下午去健身的话，外面的天气很舒服哦 😊"
```

### Step 8: 流式输出

Agent Runner 将结果写入 stdout，用标记对包裹：

```
---NANOCLAW_OUTPUT_START---
{"status":"success","result":"今天上海天气晴朗，最高温度12°C...","newSessionId":"session-abc123"}
---NANOCLAW_OUTPUT_END---
```

### Step 9: 宿主机处理输出

`container-runner.ts` 实时解析 stdout 中的标记对：

```typescript
container.stdout.on('data', (data) => {
    parseBuffer += data.toString();
    while ((startIdx = parseBuffer.indexOf(OUTPUT_START_MARKER)) !== -1) {
        const endIdx = parseBuffer.indexOf(OUTPUT_END_MARKER, startIdx);
        const parsed = JSON.parse(jsonStr);
        onOutput(parsed); // 回调：发送消息、更新会话 ID
    }
});
```

### Step 10: 发送回复

回调链：
```
onOutput(parsed)
  → channel.sendMessage(chatJid, text)
    → sock.sendMessage(jid, { text: 'Andy: 今天上海天气晴朗...' })
```

消息带有 `Andy:` 前缀，让群组用户知道这是 Agent 的回复。

### Step 11: 更新状态

```typescript
// 更新最后 Agent 交互时间戳
lastAgentTimestamp[chatJid] = missedMessages[missedMessages.length - 1].timestamp;
saveState();

// 更新会话 ID（用于下次 resume）
sessions[group.folder] = output.newSessionId;
setSession(group.folder, output.newSessionId);
```

### Step 12: 容器保持活跃

容器不会立即退出！Agent Runner 进入等待循环：

```typescript
while (true) {
    const queryResult = await runQuery(prompt, sessionId, ...);
    
    // 发出会话更新标记
    writeOutput({ status: 'success', result: null, newSessionId: sessionId });
    
    // 等待下一条 IPC 消息或关闭信号
    const nextMessage = await waitForIpcMessage();
    if (nextMessage === null) break; // _close 哨兵 → 退出
    
    prompt = nextMessage; // 新消息 → 继续循环
}
```

30 分钟无新消息后，宿主机写入 `_close` 哨兵文件，容器优雅退出。

## 状态管理全景

NanoClaw 的状态分布在多个位置，每个位置有明确的职责：

### SQLite 数据库 (store/messages.db)

```
┌───────────────────────────────────────────────────────┐
│                     SQLite 状态                        │
│                                                       │
│  router_state 表:                                     │
│  ├── last_timestamp      上次轮询的时间戳               │
│  └── last_agent_timestamp  每个群组最后处理的时间戳      │
│       {"120363...@g.us": "2026-02-23T10:30:00.000Z"}  │
│                                                       │
│  sessions 表:                                         │
│  ├── family-chat → session-abc123                     │
│  └── main       → session-xyz789                     │
│                                                       │
│  registered_groups 表:                                │
│  └── 120363...@g.us → {name: "家庭群", folder: "..."}  │
│                                                       │
│  messages 表: 完整消息历史                              │
│  scheduled_tasks 表: 定时任务定义                       │
│  task_run_logs 表: 任务执行日志                         │
└───────────────────────────────────────────────────────┘
```

### 内存状态 (index.ts 模块级变量)

```typescript
let lastTimestamp = '';                                  // 全局消息轮询游标
let sessions: Record<string, string> = {};              // 群组 → 会话ID
let registeredGroups: Record<string, RegisteredGroup> = {}; // JID → 群组配置
let lastAgentTimestamp: Record<string, string> = {};    // JID → 最后Agent处理时间
```

这些状态在启动时从 SQLite 加载，修改后立即写回。这确保了即使进程崩溃，重启后也能恢复。

### 文件系统状态

```
groups/
├── global/CLAUDE.md       ← 全局记忆（Agent 自动读写）
├── family-chat/
│   ├── CLAUDE.md          ← 群组记忆
│   ├── conversations/     ← 对话归档（compaction 时自动保存）
│   └── logs/              ← 容器运行日志
│
data/
├── sessions/family-chat/
│   └── .claude/           ← Claude SDK 会话数据（JSONL 转录）
└── ipc/family-chat/
    ├── messages/          ← 待发消息
    ├── tasks/             ← 待执行任务操作
    ├── input/             ← 宿主机 → 容器的消息管道
    └── current_tasks.json ← 任务快照（容器只读）
```

## 双游标机制

NanoClaw 使用两个时间戳游标来追踪消息处理进度：

```
时间轴: ──────────────────────────────────────→

消息:   M1    M2    M3(触发)   M4    M5(触发)
         │     │      │         │      │
         └─────┼──────┼─────────┼──────┘
               │      │         │
lastTimestamp ─┘      │         │  ← "消息循环看到了这些消息"
                      │         │
lastAgentTimestamp[g] ┘         │  ← "Agent 处理到了这里"
                                │
                                └── 下次触发时，M4也会作为上下文包含
```

- `lastTimestamp`：全局轮询游标，标记消息循环已"看到"的最新消息
- `lastAgentTimestamp[groupJid]`：每个群组的处理游标，标记 Agent 已处理到的位置

**为什么需要两个？** 因为"看到"和"处理"是两个不同的步骤。消息循环可能看到了 5 条消息，但 Agent 只处理了前 3 条（因为后 2 条没有触发词）。当后续消息包含触发词时，之前积累的未处理消息也会作为上下文一并发送给 Agent。

## 崩溃恢复机制

```typescript
// 启动时的恢复逻辑
function recoverPendingMessages(): void {
    for (const [chatJid, group] of Object.entries(registeredGroups)) {
        const pending = getMessagesSince(chatJid, lastAgentTimestamp[chatJid], ASSISTANT_NAME);
        if (pending.length > 0) {
            queue.enqueueMessageCheck(chatJid); // 重新处理
        }
    }
}
```

如果进程在处理消息期间崩溃：
1. `lastTimestamp` 已推进（消息已"看到"）
2. `lastAgentTimestamp` 可能未推进（Agent 未完成处理）
3. 重启后，`recoverPendingMessages()` 检测到差距，重新排队处理

如果 Agent 已经发送了部分回复后出错：
```typescript
if (outputSentToUser) {
    // 已经发了回复给用户，不回滚游标（避免重复发送）
    return true;
}
// 没发过回复，回滚游标让消息可以重试
lastAgentTimestamp[chatJid] = previousCursor;
```

## 小结

NanoClaw 的消息流转和状态管理有三个核心设计原则：

1. **轮询而非推送**：用简单的定时轮询替代复杂的事件驱动，降低了系统复杂度
2. **双游标追踪**：分离"已看到"和"已处理"，支持上下文追赶和崩溃恢复
3. **持久化优先**：每次状态变更立即写入 SQLite，确保崩溃后可恢复

---

**下一篇：** [04 — 容器隔离与安全模型](./04-container-isolation-and-security.md)
