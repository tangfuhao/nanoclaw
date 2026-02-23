# 09 — 容器释放与上下文恢复机制

## 核心问题

你让 Agent 写了一篇文档，写到一半你说"先这样"，Agent 完成后容器被释放。过了两小时你说"继续写那篇文档"。Agent 怎么知道之前写到了哪里？

答案涉及三层持久化机制的协作。

## 三层持久化体系

```
┌─────────────────────────────────────────────────────────────────┐
│ 第一层：文件系统 (groups/{name}/)                                 │
│                                                                 │
│ ✅ 容器释放后完整保留                                              │
│ Agent 创建的所有文件（文档、代码、笔记）都在这里                       │
│ → 下次容器挂载同一目录，文件原封不动                                  │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ 第二层：Claude Session (data/sessions/{name}/.claude/)            │
│                                                                 │
│ ✅ 容器释放后完整保留                                              │
│ Claude SDK 的会话记录（对话历史、工具调用记录、紧凑摘要）               │
│ → 下次启动容器时 SDK 使用 resume 参数恢复                           │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ 第三层：群组记忆 (groups/{name}/CLAUDE.md)                        │
│                                                                 │
│ ✅ 容器释放后完整保留                                              │
│ Agent 自己维护的长期记忆文件，类似"个人笔记"                         │
│ → 每次启动时 Claude SDK 自动加载为系统上下文                         │
└─────────────────────────────────────────────────────────────────┘
```

## 第一层：文件系统持久化

### 工作原理

```
宿主机目录结构:
groups/
├── main/           ← 主群组的工作目录
│   ├── CLAUDE.md   ← Agent 记忆
│   ├── my-doc.md   ← Agent 创建的文档
│   └── logs/
│
├── family-chat/    ← family-chat 群组
│   ├── CLAUDE.md
│   ├── report.md   ← Agent 写的报告
│   └── logs/
│
└── global/         ← 全局共享记忆 (只读)
    └── CLAUDE.md
```

当容器启动时：

```typescript
// container-runner.ts
mounts.push({
  hostPath: 'groups/family-chat/',     // 宿主机路径
  containerPath: '/workspace/group',    // 容器内路径
  readonly: false,                      // 可写
});
```

Agent 在容器内通过 `/workspace/group/` 读写文件。容器退出后，文件留在宿主机的 `groups/family-chat/`。下次容器再启动，挂载同一目录，所有文件原样还在。

**关键理解：容器是"临时访客"，文件系统是"永久住所"。**

### 你的场景

```
第一次对话: "帮我写一篇文档"
  → Agent 创建 /workspace/group/my-doc.md
  → Agent 写了 3 个章节
  → 容器退出，groups/family-chat/my-doc.md 保留

两小时后: "继续写那篇文档"
  → 新容器启动，挂载同一个 groups/family-chat/
  → Agent ls /workspace/group/ → 看到 my-doc.md
  → Agent cat my-doc.md → 读到之前写的 3 个章节
  → 继续从第 4 章开始写
```

## 第二层：Claude Session 恢复

### Session ID 的生命周期

```
首次对话 (无 sessionId):
  ┌──────────────────────────────────────────────────────┐
  │ 宿主机                                                │
  │ const sessionId = sessions['family-chat']; // undefined│
  │ → 传给容器: { sessionId: undefined, ... }              │
  └──────────────────────────────────────────────────────┘
         ↓
  ┌──────────────────────────────────────────────────────┐
  │ 容器内                                                │
  │ query({ resume: undefined, ... })                     │
  │ → SDK 创建全新会话                                      │
  │ → 收到 system/init 消息: session_id = "sess_abc123"    │
  │ → writeOutput({ newSessionId: "sess_abc123" })        │
  └──────────────────────────────────────────────────────┘
         ↓
  ┌──────────────────────────────────────────────────────┐
  │ 宿主机                                                │
  │ sessions['family-chat'] = 'sess_abc123';              │
  │ setSession('family-chat', 'sess_abc123');  // → SQLite │
  └──────────────────────────────────────────────────────┘


第二次对话 (有 sessionId):
  ┌──────────────────────────────────────────────────────┐
  │ 宿主机                                                │
  │ const sessionId = sessions['family-chat'];            │
  │   // = 'sess_abc123'                                  │
  │ → 传给容器: { sessionId: 'sess_abc123', ... }         │
  └──────────────────────────────────────────────────────┘
         ↓
  ┌──────────────────────────────────────────────────────┐
  │ 容器内                                                │
  │ query({                                               │
  │   resume: 'sess_abc123',    ← 恢复这个会话             │
  │   resumeSessionAt: undefined, ← 从最新位置继续          │
  │   ...                                                 │
  │ })                                                    │
  │ → SDK 加载 sess_abc123 的对话历史                        │
  │ → Claude 看到之前的所有对话记录                           │
  │ → 新的用户消息加入同一个会话                               │
  └──────────────────────────────────────────────────────┘
```

### Session 数据存储在哪里？

```
宿主机:
data/sessions/family-chat/.claude/
├── settings.json         ← SDK 设置
├── projects/
│   └── workspace-group/  ← 项目数据（基于 cwd 路径 hash）
│       ├── sessions/
│       │   └── sess_abc123/
│       │       ├── transcript.jsonl  ← 完整对话记录
│       │       └── ...
│       ├── sessions-index.json       ← 会话索引（含摘要）
│       └── CLAUDE.md                 ← 项目级记忆
├── skills/               ← 技能定义
└── memory/               ← Claude 自动记忆
```

容器启动时，这个目录被挂载到 `/home/node/.claude/`：

```typescript
mounts.push({
  hostPath: 'data/sessions/family-chat/.claude/',
  containerPath: '/home/node/.claude',
  readonly: false,
});
```

**Claude SDK 在容器内看到的路径和普通的 `claude` CLI 用户看到的一模一样。** 它不知道自己在容器里——对它来说，`/home/node/.claude` 就是它的 home 目录。

### `resumeSessionAt` 的作用

在容器内的查询循环中，Agent Runner 追踪每个 query 返回的最后一个 `assistant` 消息的 UUID：

```typescript
// agent-runner/src/index.ts

let resumeAt: string | undefined;

while (true) {
  const queryResult = await runQuery(prompt, sessionId, ..., resumeAt);

  if (queryResult.lastAssistantUuid) {
    resumeAt = queryResult.lastAssistantUuid;  // 记录最后的 assistant 消息位置
  }

  // 等待下一条消息...
  prompt = nextMessage;
  // 下一轮 query 会从 resumeAt 位置恢复
}
```

**`resumeSessionAt` 告诉 SDK "从这个位置继续对话"**，避免重复处理已经处理过的消息。这在单次容器生命周期内的多轮查询中尤其重要。

## 第三层：CLAUDE.md 群组记忆

### 自动加载机制

Claude Agent SDK 有一个内置行为：**启动时自动读取工作目录下的 `CLAUDE.md` 文件**，将其内容作为系统上下文的一部分提供给 Claude。

```
容器工作目录: /workspace/group/
Claude SDK 自动读取: /workspace/group/CLAUDE.md
  → 内容注入到 Claude 的系统提示中
```

### Agent 自主维护记忆

Claude 有 `auto-memory` 能力——在对话中学到的重要信息会被自动写入 CLAUDE.md：

```markdown
# groups/family-chat/CLAUDE.md

## 用户偏好
- 用户喜欢简洁的回复
- 时区: Asia/Shanghai
- 文档格式偏好: Markdown

## 进行中的项目
- 正在写"AI入门教程"，已完成1-3章
- 下次需要继续第4章
```

### 全局记忆

非 main 群组还会加载全局记忆：

```typescript
// agent-runner/src/index.ts

const globalClaudeMdPath = '/workspace/global/CLAUDE.md';
if (!containerInput.isMain && fs.existsSync(globalClaudeMdPath)) {
  globalClaudeMd = fs.readFileSync(globalClaudeMdPath, 'utf-8');
}

// 传给 SDK
query({
  options: {
    systemPrompt: {
      type: 'preset',
      preset: 'claude_code',
      append: globalClaudeMd,  // 追加到系统提示
    },
  }
});
```

## PreCompact Hook：对话归档

当 Claude SDK 的对话历史过长时，它会执行**紧凑化 (compaction)**——将旧对话压缩为摘要。NanoClaw 在紧凑化之前会归档完整对话：

```typescript
function createPreCompactHook(assistantName?: string): HookCallback {
  return async (input) => {
    const transcriptPath = input.transcript_path;

    // 读取完整对话记录
    const content = fs.readFileSync(transcriptPath, 'utf-8');
    const messages = parseTranscript(content);

    // 获取会话摘要
    const summary = getSessionSummary(sessionId, transcriptPath);
    const name = summary ? sanitizeFilename(summary) : generateFallbackName();

    // 归档到 conversations/ 目录
    const filename = `${date}-${name}.md`;
    fs.writeFileSync(
      `/workspace/group/conversations/${filename}`,
      formatTranscriptMarkdown(messages, summary, assistantName)
    );

    return {};
  };
}
```

归档后的对话保存在群组目录下，即使 session 被紧凑化（旧的详细对话被替换为摘要），原始对话仍然可以作为文件被 Agent 读取。

## 完整恢复流程图

```
    第一次对话                    容器释放               第二次对话
    ─────────                    ────────               ─────────

    container starts              container exits        container starts
         │                             │                      │
    ┌────▼────┐                        │                 ┌────▼────┐
    │ 无 session │                     │                 │ 有 session │
    │ resume:   │                      │                 │ resume:    │
    │ undefined │                      │                 │ sess_abc   │
    └────┬────┘                        │                 └────┬────┘
         │                             │                      │
    ┌────▼──────────┐                  │                 ┌────▼──────────┐
    │ SDK 创建新会话  │                  │                 │ SDK 恢复旧会话  │
    │ → sess_abc    │                  │                 │ → 加载对话历史  │
    │               │                  │                 │               │
    │ Claude 读取:   │                  │                 │ Claude 读取:   │
    │ - CLAUDE.md   │                  │                 │ - CLAUDE.md   │ 
    │ - (空目录)     │                  │                 │ - my-doc.md   │ ← 上次写的文档还在！
    └────┬──────────┘                  │                 └────┬──────────┘
         │                             │                      │
    Agent 写 my-doc.md                 │                 Claude 看到之前的对话:
    Agent 更新 CLAUDE.md               │                 "用户让我写文档，我写到第3章"
         │                             │                      │
    ┌────▼──────────┐            ┌─────▼──────┐         ┌────▼──────────┐
    │ 保存到宿主机:   │            │ 保留在宿主机: │         │ Claude 回复:    │
    │ - sess_abc    │──────────>│ - sess_abc   │────────>│ "好的，我继续   │
    │   → SQLite    │            │   → SQLite   │         │  写第4章"       │
    │ - my-doc.md   │            │ - my-doc.md  │         └───────────────┘
    │   → groups/   │            │   → groups/  │
    │ - CLAUDE.md   │            │ - CLAUDE.md  │
    │ - .claude/    │            │ - .claude/   │
    │   → data/     │            │   → data/    │
    └───────────────┘            └──────────────┘
```

## Claude 具体能"记住"什么？

| 信息类型 | 存储位置 | 恢复方式 | 持久性 |
|---------|---------|---------|--------|
| 对话历史 | `.claude/projects/*/sessions/` | `resume` 参数 | 持久（紧凑化后变摘要） |
| 文件内容 | `groups/{name}/` | Agent 重新 `cat` 文件 | 永久 |
| 用户偏好 | `groups/{name}/CLAUDE.md` | SDK 自动加载 | 永久（Agent 自维护） |
| 全局配置 | `groups/global/CLAUDE.md` | 通过 `append` 注入 | 永久 |
| 自动记忆 | `.claude/memory/` | SDK 内置机制 | 永久 |
| 归档对话 | `groups/{name}/conversations/` | Agent 可主动查阅 | 永久 |

## 边界情况

### Session 文件损坏或丢失

如果 `.claude/` 目录被意外删除，`sessions` 表中的 session ID 仍然存在，但 SDK 找不到对应的会话文件。这时 SDK 会报错或创建新会话。宿主机检测到错误后会更新 session ID。

### 紧凑化后的上下文损失

当对话非常长时，SDK 会将旧对话压缩为摘要。这意味着 Claude 不再记得"3 小时前你说的那句原话"，但它记得摘要内容。通过 PreCompact Hook 归档的原始对话可以被 Agent 在需要时重新查阅。

### 跨群组的记忆隔离

每个群组有独立的 `.claude/` 目录和 `groups/{name}/` 目录。群组 A 的 Agent 无法看到群组 B 的文件或对话历史。唯一的共享渠道是 `groups/global/CLAUDE.md`（只读）。

---

**下一篇：** [10 — 边界情况与异常处理](./10-edge-cases-and-error-handling.md)
