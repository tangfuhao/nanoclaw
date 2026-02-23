# 08 — NanoClaw 与 Claude Code 交互全流程

## 概述

NanoClaw 宿主机本身**不直接调用 Claude API**。它通过 `docker run` 启动一个 Linux 容器，容器内运行 `@anthropic-ai/claude-code` (即 Claude Agent SDK)，SDK 负责与 Anthropic API 通信。宿主机和容器的关系如同"经理"和"外包员工"——经理下发任务，员工在隔离房间里独立完成，结果通过窗口传回来。

## 完整调用链

```
用户消息  →  WhatsApp  →  宿主机 Node.js  →  docker run  →  容器内 agent-runner
                                                                    ↓
                                                          Claude Agent SDK query()
                                                                    ↓
                                                          Anthropic Claude API
                                                                    ↓
                                                          Claude 思考 + 调用工具
                                                                    ↓
                                                          stdout 标记输出
                                                                    ↓
                                              宿主机解析 stdout  ←  容器进程
                                                     ↓
                                              WhatsApp 发送给用户
```

## 第一步：宿主机准备工作

当宿主机决定启动一个 Agent 容器时（因为有新消息或定时任务触发），`runAgent()` 函数执行如下准备：

```typescript
// index.ts → runAgent()

// 1. 从 SQLite 读取该群组的 session ID（用于恢复上下文）
const sessionId = sessions[group.folder];

// 2. 生成数据快照供容器只读访问
writeTasksSnapshot(group.folder, isMain, tasks);
writeGroupsSnapshot(group.folder, isMain, availableGroups);

// 3. 构建 ContainerInput 对象
const input: ContainerInput = {
  prompt,          // XML 格式化的消息
  sessionId,       // 上次的 session ID（恢复上下文用）
  groupFolder,     // 群组文件夹名
  chatJid,         // WhatsApp 群组 JID
  isMain,          // 是否为主群组
  assistantName,   // 助手名字
};

// 4. 调用 container-runner
const output = await runContainerAgent(group, input, ...);
```

## 第二步：容器启动

`runContainerAgent()` 执行以下操作：

### 2.1 构建挂载卷

```typescript
// container-runner.ts → buildVolumeMounts()

const mounts = [
  // 群组工作目录 (可写)
  { hostPath: 'groups/family-chat/', containerPath: '/workspace/group' },

  // Claude 会话数据 (可写)
  { hostPath: 'data/sessions/family-chat/.claude/', containerPath: '/home/node/.claude' },

  // IPC 目录 (可写)
  { hostPath: 'data/ipc/family-chat/', containerPath: '/workspace/ipc' },

  // Agent runner 源码 (可写，支持自定义)
  { hostPath: 'data/sessions/family-chat/agent-runner-src/', containerPath: '/app/src' },

  // 全局记忆 (只读，非 main)
  { hostPath: 'groups/global/', containerPath: '/workspace/global', readonly: true },
];
```

### 2.2 生成 Docker 命令

```bash
docker run -i --rm \
  --name nanoclaw-family-chat-1708688400000 \
  -e TZ=Asia/Shanghai \
  --user 501:20 \                              # 宿主机 UID:GID
  -v groups/family-chat:/workspace/group \     # 可写
  -v data/sessions/.claude:/home/node/.claude \ # 可写
  -v data/ipc/family-chat:/workspace/ipc \     # 可写
  -v groups/global:/workspace/global:ro \      # 只读
  nanoclaw-agent:latest
```

关键参数说明：
- **`-i`**：保持 stdin 开放（用于注入 JSON）
- **`--rm`**：容器退出时自动清理
- **`--user`**：以宿主机用户运行，确保文件权限一致

### 2.3 通过 stdin 传递密钥

```typescript
// container-runner.ts

input.secrets = readSecrets();
//   → { ANTHROPIC_API_KEY: 'sk-...', CLAUDE_CODE_OAUTH_TOKEN: '...' }

container.stdin.write(JSON.stringify(input));
container.stdin.end();

delete input.secrets;  // 立即从内存清除，不出现在日志中
```

**为什么用 stdin 而不是环境变量？**

| 方法 | 安全性 | 原因 |
|------|--------|------|
| 环境变量 `-e KEY=val` | ❌ 不安全 | `docker inspect` 可看到 |
| 挂载密钥文件 | ❌ 不安全 | 容器内进程可读取 |
| **stdin 传递** | ✅ 安全 | 读完就没了，无持久化 |

## 第三步：容器内入口点

容器启动后执行 `entrypoint.sh`：

```bash
#!/bin/bash
set -e

# 1. 重新编译 agent-runner（因为源码可能被群组自定义了）
cd /app && npx tsc --outDir /tmp/dist 2>&1 >&2

# 2. 链接 node_modules（避免重新安装）
ln -s /app/node_modules /tmp/dist/node_modules

# 3. 设为只读（防止运行时篡改）
chmod -R a-w /tmp/dist

# 4. 从 stdin 读取 JSON 到临时文件
cat > /tmp/input.json

# 5. 启动 agent-runner
node /tmp/dist/index.js < /tmp/input.json
```

## 第四步：Agent Runner 启动 Claude SDK

### 4.1 解析输入

```typescript
// container/agent-runner/src/index.ts → main()

const stdinData = await readStdin();
const containerInput = JSON.parse(stdinData);
fs.unlinkSync('/tmp/input.json');  // 立即删除含密钥的临时文件

// 密钥放入 SDK 专用环境变量（不污染 process.env）
const sdkEnv = { ...process.env };
for (const [key, value] of Object.entries(containerInput.secrets || {})) {
  sdkEnv[key] = value;
}
```

### 4.2 创建 MessageStream

```typescript
// MessageStream 是一个 AsyncIterable<SDKUserMessage>
// 它的关键作用：让 SDK 认为"还有更多用户消息可能到来"
// 这样 SDK 不会设 isSingleUserTurn=true（否则 Agent Teams 子代理会被截断）

const stream = new MessageStream();
stream.push(prompt);  // 第一条消息
```

**`isSingleUserTurn` 的问题：**

当 Claude Agent SDK 认为只有一个用户消息时，它会设置 `isSingleUserTurn=true`。在 Agent Teams 场景下，这会导致子代理（subagent）在完成任务前被强制终止。`MessageStream` 作为 `AsyncIterable`，让 SDK 始终认为"可能还有后续消息"，从而避免了这个问题。

### 4.3 调用 Claude SDK query()

```typescript
for await (const message of query({
  prompt: stream,            // AsyncIterable — 保持对话通道开放
  options: {
    cwd: '/workspace/group', // 工作目录（Agent 在这里读写文件）
    resume: sessionId,       // 恢复之前的会话 ← 关键！
    resumeSessionAt: resumeAt, // 从哪个消息继续
    systemPrompt: {          // 系统提示（含全局记忆）
      type: 'preset',
      preset: 'claude_code',
      append: globalClaudeMd,
    },
    allowedTools: [          // 允许的工具集
      'Bash', 'Read', 'Write', 'Edit',
      'Glob', 'Grep', 'WebSearch', 'WebFetch',
      'Task', 'TeamCreate', 'SendMessage',
      'mcp__nanoclaw__*',    // NanoClaw MCP 工具
    ],
    env: sdkEnv,             // 密钥在这里传给 SDK
    permissionMode: 'bypassPermissions',
    mcpServers: {
      nanoclaw: {            // 在容器内启动 MCP 服务器
        command: 'node',
        args: [mcpServerPath],
        env: {
          NANOCLAW_CHAT_JID: chatJid,
          NANOCLAW_GROUP_FOLDER: groupFolder,
          NANOCLAW_IS_MAIN: isMain ? '1' : '0',
        },
      },
    },
    hooks: {
      PreCompact: [createPreCompactHook()],     // 压缩前归档对话
      PreToolUse: [{ matcher: 'Bash', hooks: [createSanitizeBashHook()] }],
      // ↑ 每次 Bash 调用前注入 "unset ANTHROPIC_API_KEY" 防止密钥泄漏
    },
  }
})) {
  // 处理每一条 SDK 消息...
}
```

### 4.4 SDK 消息流处理

```typescript
for await (const message of query({ ... })) {
  // 消息类型:
  // type='system', subtype='init'  → 会话初始化，获取 sessionId
  // type='assistant'               → Claude 的回复（含 uuid）
  // type='result'                  → 一个完整的结果，需要输出

  if (message.type === 'system' && message.subtype === 'init') {
    newSessionId = message.session_id;  // ← 保存会话 ID
  }

  if (message.type === 'result') {
    writeOutput({                      // ← 写到 stdout
      status: 'success',
      result: textResult || null,
      newSessionId
    });
  }
}
```

## 第五步：输出回传宿主机

### 5.1 标记协议

容器通过 stdout 输出，使用特殊标记分隔多个结果：

```
---NANOCLAW_OUTPUT_START---
{"status":"success","result":"天气晴朗，12°C","newSessionId":"sess_abc123"}
---NANOCLAW_OUTPUT_END---
```

一个容器生命周期内可能输出多个结果（多轮 query）。

### 5.2 宿主机流式解析

```typescript
// container-runner.ts

container.stdout.on('data', (data) => {
  parseBuffer += data.toString();

  while ((startIdx = parseBuffer.indexOf(OUTPUT_START_MARKER)) !== -1) {
    const endIdx = parseBuffer.indexOf(OUTPUT_END_MARKER, startIdx);
    if (endIdx === -1) break;  // 不完整，等更多数据

    const jsonStr = parseBuffer.slice(startIdx + ..., endIdx).trim();
    const parsed = JSON.parse(jsonStr);

    if (parsed.newSessionId) {
      newSessionId = parsed.newSessionId;  // 保存最新 session ID
    }

    onOutput(parsed);  // → 回调到 index.ts → 发送给 WhatsApp
  }
});
```

### 5.3 最终发送

```typescript
// index.ts → processGroupMessages() → onOutput 回调

const output = await runAgent(group, prompt, chatJid, async (result) => {
  if (result.result) {
    const text = raw.replace(/<internal>[\s\S]*?<\/internal>/g, '').trim();
    if (text) {
      await channel.sendMessage(chatJid, text);  // → WhatsApp
    }
  }

  if (result.status === 'success') {
    queue.notifyIdle(chatJid);  // 标记容器空闲，开始计时
  }
});
```

## 第六步：会话保持与多轮对话

容器不会在一次 query 完成后立即退出。它进入一个**查询循环**：

```typescript
// container/agent-runner/src/index.ts → main()

while (true) {
  // 1. 执行 query
  const queryResult = await runQuery(prompt, sessionId, ...);

  // 2. 更新 session 追踪
  sessionId = queryResult.newSessionId;
  resumeAt = queryResult.lastAssistantUuid;

  // 3. 如果收到关闭信号，退出
  if (queryResult.closedDuringQuery) break;

  // 4. 通知宿主机 session 更新
  writeOutput({ status: 'success', result: null, newSessionId: sessionId });

  // 5. 等待下一条 IPC 消息
  const nextMessage = await waitForIpcMessage();
  if (nextMessage === null) break;  // 收到 _close 信号

  // 6. 用新消息开始下一轮 query
  prompt = nextMessage;
}
```

**这个循环是"管道模式"的核心**——一个容器可以处理多轮对话，而不需要每次都冷启动。

## 安全屏障总结

```
                       ┌─────────────────────────────┐
                       │        宿主机 (可信域)        │
                       │                             │
                       │  secrets 只在内存中短暂存在     │
                       │  delete input.secrets 立即清除 │
                       └──────────────┬──────────────┘
                                      │
                              stdin (JSON, 含密钥)
                              一次性传输，不可重放
                                      │
                       ┌──────────────▼──────────────┐
                       │        容器 (非可信域)        │
                       │                             │
                       │  sdkEnv: 密钥只给 SDK        │
                       │  process.env: 不含密钥       │
                       │                             │
                       │  PreToolUse Hook:           │
                       │  每次 Bash 执行前自动:        │
                       │  unset ANTHROPIC_API_KEY     │
                       │  unset CLAUDE_CODE_OAUTH_TOKEN│
                       │                             │
                       │  /tmp/input.json 读后即删     │
                       └─────────────────────────────┘
```

Claude 可以执行 Bash 命令（它需要这个能力来工作），但通过 Hook 机制，每个 Bash 命令执行前都会自动 `unset` 敏感环境变量，确保即使被提示注入攻击，攻击者也无法通过 `echo $ANTHROPIC_API_KEY` 这样的命令获取密钥。

## 工具能力一览

容器内的 Claude 拥有以下工具：

| 类别 | 工具 | 来源 |
|------|------|------|
| 内置 | Bash, Read, Write, Edit, Glob, Grep | Claude Agent SDK |
| 网络 | WebSearch, WebFetch | Claude Agent SDK |
| 团队 | Task, TeamCreate, SendMessage | Claude Agent SDK (Agent Teams) |
| 其他 | TodoWrite, ToolSearch, Skill, NotebookEdit | Claude Agent SDK |
| 自定义 | mcp__nanoclaw__send_message | NanoClaw MCP Server |
| 自定义 | mcp__nanoclaw__schedule_task | NanoClaw MCP Server |
| 自定义 | mcp__nanoclaw__list_tasks | NanoClaw MCP Server |
| 自定义 | mcp__nanoclaw__register_group | NanoClaw MCP Server |
| Bash 扩展 | agent-browser (浏览器自动化) | 通过 Bash 调用全局命令 |

---

**下一篇：** [09 — 容器释放与上下文恢复机制](./09-session-resume-and-context.md)
