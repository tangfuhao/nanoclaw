# 05 — 工具系统与技能扩展

## Agent 的工具体系

Agent 的能力取决于它拥有的工具。没有工具的 Agent 只是一个聊天机器人。NanoClaw 中 Agent 的工具分为三类：

```
┌──────────────────────────────────────────────────────────────┐
│                     Agent 工具全景                             │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐     │
│  │ 第一类: Claude Agent SDK 内置工具                     │     │
│  │                                                     │     │
│  │  Bash      — 执行 Shell 命令（容器内安全）             │     │
│  │  Read      — 读取文件                                │     │
│  │  Write     — 创建/覆写文件                            │     │
│  │  Edit      — 编辑文件内容                             │     │
│  │  Glob      — 按模式查找文件                           │     │
│  │  Grep      — 搜索文件内容                             │     │
│  │  WebSearch — 网络搜索                                │     │
│  │  WebFetch  — 获取网页内容                             │     │
│  │  Task      — 启动子 Agent（同步/异步）                 │     │
│  │  TodoWrite — 任务列表管理                             │     │
│  │                                                     │     │
│  │  这些工具由 SDK 实现，Agent Runner 只需在                │     │
│  │  allowedTools 列表中声明即可使用                        │     │
│  └─────────────────────────────────────────────────────┘     │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐     │
│  │ 第二类: MCP 工具（NanoClaw 自定义）                    │     │
│  │                                                     │     │
│  │  send_message   — 发送消息到群组                      │     │
│  │  schedule_task  — 创建定时任务                        │     │
│  │  list_tasks     — 列出任务                           │     │
│  │  pause_task     — 暂停任务                           │     │
│  │  resume_task    — 恢复任务                           │     │
│  │  cancel_task    — 取消任务                           │     │
│  │  register_group — 注册新群组（仅 Main）                │     │
│  │                                                     │     │
│  │  这些工具通过 MCP 协议提供，实现在                      │     │
│  │  container/agent-runner/src/ipc-mcp-stdio.ts          │     │
│  └─────────────────────────────────────────────────────┘     │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐     │
│  │ 第三类: 通过 Bash 调用的外部工具                       │     │
│  │                                                     │     │
│  │  agent-browser — 浏览器自动化                         │     │
│  │    用法: agent-browser navigate https://example.com   │     │
│  │    用法: agent-browser screenshot page.png            │     │
│  │                                                     │     │
│  │  git, curl, python — 容器内预装的命令行工具             │     │
│  │                                                     │     │
│  │  这些工具不需要特殊注册，Agent 通过 Bash 工具调用         │     │
│  └─────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────┘
```

## MCP 协议：Agent 的工具扩展标准

### 什么是 MCP？

**MCP (Model Context Protocol)** 是 Anthropic 提出的开放协议，让 LLM 应用能够与外部工具和数据源交互。可以把它理解为"AI 的 USB 接口"——任何实现了 MCP 协议的服务都可以被 AI 使用。

### MCP 在 NanoClaw 中的角色

NanoClaw 使用 MCP 来提供自定义工具（发消息、管理任务等）：

```
┌─────────────────────┐     ┌──────────────────────┐
│   Claude Agent SDK   │     │  NanoClaw MCP 服务器  │
│   (工具消费者)        │     │  (工具提供者)          │
│                     │     │                      │
│  Agent 调用:        │     │  提供工具:             │
│  mcp__nanoclaw__    │ ←→  │  send_message         │
│  schedule_task      │     │  schedule_task         │
│                     │     │  list_tasks           │
│  底层通信:          │     │  ...                   │
│  JSON-RPC over     │     │                      │
│  stdin/stdout      │     │  实际操作:             │
│                     │     │  写 IPC 文件到磁盘      │
└─────────────────────┘     └──────────────────────┘
```

### MCP 服务器的实现

NanoClaw 的 MCP 服务器（`ipc-mcp-stdio.ts`）是一个标准的 MCP stdio 服务器：

```typescript
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { StdioServerTransport } from '@modelcontextprotocol/sdk/server/stdio.js';
import { z } from 'zod';

const server = new McpServer({ name: 'nanoclaw', version: '1.0.0' });

// 注册工具
server.tool(
    'send_message',                              // 工具名
    '发送消息到群组...',                            // 描述（LLM 据此决定何时使用）
    { text: z.string(), sender: z.string().optional() }, // 输入 schema (Zod)
    async (args) => {                            // 处理函数
        writeIpcFile(MESSAGES_DIR, {
            type: 'message',
            chatJid,          // 从环境变量获取（群组身份）
            text: args.text,
        });
        return { content: [{ type: 'text', text: 'Message sent.' }] };
    },
);

// 启动 stdio 传输
const transport = new StdioServerTransport();
await server.connect(transport);
```

**关键点：**
- 工具描述必须清晰，因为 LLM 根据描述决定何时使用工具
- 输入使用 Zod schema 定义，SDK 自动验证
- 工具通过写 IPC 文件与宿主机通信（不直接操作数据库）
- 群组身份从环境变量注入，不可被容器内代码修改

### 为什么用 stdio 而不是 in-process？

NanoClaw 选择 stdio MCP 服务器而非 in-process SDK 服务器，有一个重要原因：

```
选项 A: in-process MCP (createSdkMcpServer)
  Agent → MCP Server (同一进程内)
  问题: Agent Teams 的子 Agent 无法继承此 MCP 服务器

选项 B: stdio MCP (StdioServerTransport) ✓
  Agent → spawn → MCP Server (独立进程)
  Agent Teams 子 Agent → spawn → 同一 MCP Server
  优点: 所有子 Agent 都能使用 schedule_task/send_message
```

当 Agent 使用 Agent Teams 功能时，Leader Agent 会派生多个 Sub-Agent。如果 MCP 服务器是 in-process 的，子 Agent 无法访问它。而 stdio 模式下，MCP 服务器配置会被子 Agent 自动继承。

## IPC 通信：容器与宿主机的桥梁

### 双向通信设计

```
                    ┌───────────────────┐
                    │      宿主机        │
                    │                   │
         ┌─────────┤  IPC 监视器       │──────────┐
         │         │  (每秒轮询)        │          │
         │         └───────────────────┘          │
         │                  ↑                     │
         │                  │ 读取                 │
         ▼                  │                     ▼
    ┌─────────┐    ┌────────────────┐    ┌──────────────┐
    │ 执行操作  │    │ data/ipc/{grp}/ │    │  写入消息     │
    │ (发消息、 │    │ ├── messages/   │    │ (管道模式)    │
    │  创建任务)│    │ ├── tasks/      │    │              │
    └─────────┘    │ ├── input/      │    └──────────────┘
                    │ └── _close      │          │
                    └────────────────┘          │
                           ↑                    │
                           │ 写入                │ 写入
                           │                    │
                    ┌──────┴────────────────────┴─┐
                    │          容器内               │
                    │                             │
                    │  MCP Server                  │
                    │  (send_message → messages/)  │
                    │  (schedule_task → tasks/)    │
                    │                             │
                    │  Agent Runner                │
                    │  (读取 input/ 中的新消息)      │
                    │  (检测 _close 哨兵文件)        │
                    └─────────────────────────────┘
```

### 原子性写入

IPC 使用"写临时文件 + rename"的原子性策略：

```typescript
function writeIpcFile(dir: string, data: object): string {
    const filepath = path.join(dir, `${Date.now()}-${randomId()}.json`);
    const tempPath = `${filepath}.tmp`;
    
    fs.writeFileSync(tempPath, JSON.stringify(data, null, 2)); // 1. 写临时文件
    fs.renameSync(tempPath, filepath);                          // 2. 原子重命名
    
    return filename;
}
```

为什么不直接写？因为宿主机的 IPC 监视器可能在文件写入一半时读到它，导致 JSON 解析错误。`rename` 是操作系统级的原子操作，要么完全完成，要么完全不发生。

## 技能系统：一种新的软件扩展范式

### 传统扩展 vs 技能扩展

**传统方式（添加功能）：**
```
核心代码 → 添加 Telegram 模块 → 代码膨胀
          → 添加 Discord 模块 → 代码更膨胀
          → 添加 Slack 模块   → 维护噩梦
```

**NanoClaw 的方式（技能转换）：**
```
核心代码 → 运行 /add-telegram 技能 → 代码被改写为 Telegram 版本
         → 代码仍然简洁，没有多余的 WhatsApp 代码
```

### 技能是什么？

技能文件（`.claude/skills/*/SKILL.md`）是给 Claude Code 的指令文档，教它如何转换代码库。技能不是可执行代码，而是自然语言描述的操作步骤。

例如 `/add-telegram` 技能的结构：

```
.claude/skills/add-telegram/
├── SKILL.md                   ← 主指令文件
├── manifest.yaml              ← 新文件和修改规则
├── add/
│   └── src/channels/
│       └── telegram.ts        ← 新增的 Telegram 通道实现
├── modify/
│   ├── src/config.ts          ← 配置修改意图
│   ├── src/config.ts.intent.md
│   ├── src/index.ts           ← 入口修改意图
│   └── src/index.ts.intent.md
└── tests/
    └── telegram.test.ts       ← 测试文件
```

### manifest.yaml 解析

```yaml
name: add-telegram
description: Add Telegram as a messaging channel
version: 1.0.0

add:
  - src/channels/telegram.ts      # 新增文件

modify:
  - src/config.ts                 # 修改现有文件
  - src/index.ts

dependencies:
  - telegraf                      # 新增 npm 依赖
```

### 意图文件 (.intent.md)

意图文件描述了如何修改现有代码，而不是直接提供补丁。这让修改能适应代码库的当前状态：

```markdown
# Intent: Modify src/config.ts

Add Telegram configuration constants:
- TELEGRAM_BOT_TOKEN from environment
- TELEGRAM_TRIGGER_PATTERN for Telegram messages
```

Claude Code 读取意图后，会智能地将修改应用到代码中——即使代码库已经被用户自定义过。

### 为什么技能优于插件？

| 特性 | 传统插件 | NanoClaw 技能 |
|------|---------|-------------|
| 代码组织 | 并列模块，都在运行时加载 | 转换代码库，只保留需要的代码 |
| 性能 | 加载所有插件 | 只有实际使用的代码 |
| 复杂度 | 随插件增多而增长 | 始终保持简洁 |
| 自定义 | 受限于插件接口 | 完全自由修改 |
| 冲突 | 插件间可能冲突 | 不存在（只有一套代码） |
| 依赖 | 累积所有插件的依赖 | 只有当前功能的依赖 |

## Hook 系统：控制 Agent 行为

NanoClaw 使用 Claude Agent SDK 的 Hook 系统来修改 Agent 的行为。Hook 是在特定事件发生时自动执行的回调函数。

### PreCompact Hook：对话归档

当对话历史过长时，Claude SDK 会自动压缩（compaction）。NanoClaw 在压缩前归档完整对话：

```typescript
function createPreCompactHook(assistantName?: string): HookCallback {
    return async (input) => {
        // 读取完整转录
        const content = fs.readFileSync(transcriptPath, 'utf-8');
        const messages = parseTranscript(content);
        
        // 归档为 Markdown
        const markdown = formatTranscriptMarkdown(messages, summary, assistantName);
        fs.writeFileSync(
            `/workspace/group/conversations/${date}-${name}.md`,
            markdown
        );
        
        return {}; // 允许 compaction 继续
    };
}
```

归档后，对话历史以 Markdown 形式保存在群组文件夹中，Agent 可以随时回顾。

### PreToolUse Hook：Bash 安全防护

在每个 Bash 命令执行前，自动注入 `unset` 命令清除敏感环境变量：

```typescript
function createSanitizeBashHook(): HookCallback {
    return async (input) => {
        const command = input.tool_input?.command;
        // 原始命令: "echo $ANTHROPIC_API_KEY"
        // 修改后:   "unset ANTHROPIC_API_KEY CLAUDE_CODE_OAUTH_TOKEN 2>/dev/null; echo $ANTHROPIC_API_KEY"
        return {
            hookSpecificOutput: {
                hookEventName: 'PreToolUse',
                updatedInput: { command: unsetPrefix + command },
            },
        };
    };
}
```

### 可用的 Hook 事件

| Hook 事件 | 触发时机 | NanoClaw 的用法 |
|-----------|---------|---------------|
| PreToolUse | 工具执行前 | Bash 命令凭证清理 |
| PostToolUse | 工具执行后 | （未使用） |
| PreCompact | 对话压缩前 | 归档完整对话 |
| SessionStart | 会话开始 | （未使用） |
| SessionEnd | 会话结束 | （未使用） |
| SubagentStart | 子 Agent 启动 | （未使用） |
| SubagentStop | 子 Agent 停止 | （未使用） |

## Agent Teams：多 Agent 协作

NanoClaw 支持 Agent Teams（Agent 集群），多个 Agent 协作完成复杂任务：

```
用户: "@Andy 调研一下最近的 AI 框架，分别分析 LangChain、LlamaIndex 和 CrewAI"

Leader Agent ──→ 创建 Team
├── Researcher A: "调研 LangChain"
├── Researcher B: "调研 LlamaIndex"
└── Researcher C: "调研 CrewAI"

三个子 Agent 并行工作，完成后向 Leader 报告
Leader 汇总所有研究结果，发送给用户
```

在 NanoClaw 的 Agent Runner 中，通过在 `allowedTools` 中包含 `TeamCreate`、`TeamDelete`、`SendMessage` 工具来启用此功能。同时，在 Claude 会话设置中开启：

```json
{
    "env": {
        "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
    }
}
```

**关键技术细节：** Agent Teams 需要 `isSingleUserTurn = false`（即 prompt 是 AsyncIterable 而非字符串），否则 Leader 的第一个结果会触发 stdin 关闭，导致所有子 Agent 被强制终止。NanoClaw 通过 `MessageStream` 类保持 stdin 打开。

## 技能系统的设计哲学

NanoClaw 的技能系统体现了一种全新的软件扩展理念：

1. **AI 即构建工具**：不是人类写插件，而是 AI 根据指令转换代码
2. **代码即配置**：不需要 YAML/TOML 配置文件，直接修改代码
3. **简洁优于通用**：每个安装只包含实际需要的功能
4. **贡献知识而非代码**：技能贡献的是"如何做"的知识，而非具体的代码补丁

这种理念的核心假设是：AI 足够聪明，可以理解自然语言描述的修改意图，并将其正确地应用到代码库中。随着 AI 能力的提升，这种范式会越来越强大。

---

**下一篇：** [06 — Claude Agent SDK 深度剖析](./06-claude-agent-sdk-internals.md)
