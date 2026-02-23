# 10 — 边界情况与异常处理

## 场景 1：并发容器已满，新消息到来

### 背景

NanoClaw 默认允许最多 **5 个容器** 同时运行（`MAX_CONCURRENT_CONTAINERS=5`）。当所有槽位都被占用时，新的群组消息如何处理？

### 完整流程

```
当前状态:
  容器 1: main         → 正在处理消息
  容器 2: family-chat   → 正在运行定时任务
  容器 3: work-team     → 正在处理消息
  容器 4: study-group   → 正在处理消息
  容器 5: friends       → 正在处理消息
  activeCount = 5

此时 gaming-group 有新消息到来:
```

```typescript
// group-queue.ts → enqueueMessageCheck()

enqueueMessageCheck('gaming-group-jid') {
  const state = this.getGroup('gaming-group-jid');

  // 检查1: 该群组是否已有活跃容器？
  if (state.active) {
    // 有活跃容器 → 标记"有待处理消息"，容器完成后会自动处理
    state.pendingMessages = true;
    return;
  }

  // 检查2: 是否达到并发上限？
  if (this.activeCount >= MAX_CONCURRENT_CONTAINERS) {  // 5 >= 5
    // ✅ 达到上限！
    state.pendingMessages = true;            // 标记有消息等待
    this.waitingGroups.push('gaming-group'); // 加入等待队列
    // 不启动新容器，消息在 SQLite 中安全保存
    return;
  }

  // 低于上限才会真正启动容器
  this.runForGroup('gaming-group-jid', 'messages');
}
```

### 消息不会丢失

消息已经被存入 SQLite（在 WhatsApp 收到的时候就存了），等待队列只是一个"提醒"机制。当某个容器完成后：

```typescript
// group-queue.ts → drainGroup() → drainWaiting()

// 容器 3 (work-team) 完成了工作
finally {
  state.active = false;
  this.activeCount--;  // 5 → 4
  this.drainGroup(groupJid);
}

// drainGroup → drainWaiting
private drainWaiting(): void {
  while (
    this.waitingGroups.length > 0 &&
    this.activeCount < MAX_CONCURRENT_CONTAINERS  // 4 < 5
  ) {
    const nextJid = this.waitingGroups.shift()!;  // → 'gaming-group-jid'
    const state = this.getGroup(nextJid);

    // 优先处理任务（任务不会被重新发现，消息可以）
    if (state.pendingTasks.length > 0) {
      this.runTask(nextJid, state.pendingTasks.shift()!);
    } else if (state.pendingMessages) {
      this.runForGroup(nextJid, 'drain');  // 启动新容器处理消息
    }
  }
}
```

### 排队期间新消息不断到来

```
t=0s   gaming-group 消息 A → 存入 SQLite → 入队等待
t=10s  gaming-group 消息 B → 存入 SQLite → 已在等待队列中，标记 pendingMessages 即可
t=20s  gaming-group 消息 C → 存入 SQLite → 同上
t=30s  容器槽位释放 → 启动 gaming-group 容器
       → processGroupMessages 从 SQLite 读取 lastAgentTimestamp 之后的所有消息
       → 消息 A、B、C 都会被读取并格式化为 prompt
```

**关键理解：SQLite 是消息的权威来源，GroupQueue 只是调度协调器。**

## 场景 2：同一群组消息到达时容器正在运行

这是"管道模式"——消息被直接注入到活跃容器中，而不是启动新容器。

```
状态: family-chat 容器正在运行（处理消息 A）

新消息 B 到达:
```

```typescript
// index.ts → startMessageLoop()

// 尝试管道发送到活跃容器
if (queue.sendMessage(chatJid, formatted)) {
  // ✅ 成功! 消息通过 IPC 文件注入容器
  lastAgentTimestamp[chatJid] = messages[messages.length - 1].timestamp;
  saveState();
} else {
  // 容器不接受消息（可能是任务容器或已关闭）
  queue.enqueueMessageCheck(chatJid);  // 排队等新容器
}
```

```typescript
// group-queue.ts → sendMessage()

sendMessage(groupJid: string, text: string): boolean {
  const state = this.getGroup(groupJid);

  // 三个条件必须同时满足:
  if (!state.active) return false;        // 必须有活跃容器
  if (!state.groupFolder) return false;    // 必须知道 IPC 路径
  if (state.isTaskContainer) return false; // 不能是任务容器

  // 写 IPC 文件到容器的 input/ 目录
  const inputDir = path.join(DATA_DIR, 'ipc', state.groupFolder, 'input');
  fs.writeFileSync(filepath, JSON.stringify({ type: 'message', text }));

  return true;
}
```

**为什么任务容器不接受管道消息？** 任务容器运行的是定时任务的 prompt，注入用户消息会打乱任务逻辑。任务容器完成后，用户消息会被新的消息容器处理。

## 场景 3：容器超时

### 硬超时 vs 空闲超时

```
两种超时机制:

空闲超时 (IDLE_TIMEOUT = 30分钟):
  最后一次输出后 30 分钟无活动
  → queue.closeStdin() 写 _close 哨兵文件
  → 容器优雅退出

硬超时 (CONTAINER_TIMEOUT，默认 30 分钟):
  容器存活时间超过阈值
  → docker stop（给容器 10s 优雅关闭时间）
  → 如果仍未退出 → SIGKILL
```

```typescript
// container-runner.ts

// 硬超时至少要比空闲超时大 30s，给 _close 信号足够的处理时间
const timeoutMs = Math.max(configTimeout, IDLE_TIMEOUT + 30_000);

let timeout = setTimeout(killOnTimeout, timeoutMs);

// 每次有输出时重置硬超时
const resetTimeout = () => {
  clearTimeout(timeout);
  timeout = setTimeout(killOnTimeout, timeoutMs);
};
```

### 超时后的处理

```typescript
container.on('close', (code) => {
  if (timedOut) {
    if (hadStreamingOutput) {
      // 情况 A: 已经发过输出，超时只是空闲清理
      // → 视为成功
      resolve({ status: 'success', result: null, newSessionId });
    } else {
      // 情况 B: 从没产生过输出，超时说明出了问题
      // → 视为错误
      resolve({ status: 'error', result: null, error: 'Container timed out' });
    }
  }
});
```

## 场景 4：容器崩溃后的消息游标回滚

```
时间线:
t=0s   收到消息 A, B, C
t=1s   lastAgentTimestamp 推进到消息 C 的时间戳
       → 保存到 SQLite (避免其他逻辑重复处理)
t=2s   容器启动，开始处理
t=5s   容器崩溃！没有任何输出

处理:
```

```typescript
// index.ts → processGroupMessages()

// 预先推进游标
const previousCursor = lastAgentTimestamp[chatJid] || '';
lastAgentTimestamp[chatJid] = missedMessages[missedMessages.length - 1].timestamp;
saveState();

// ... 容器运行 ...

if (output === 'error' || hadError) {
  if (outputSentToUser) {
    // 已经发过回复了 → 不回滚（防止发重复消息）
    return true;
  }
  // 没发过回复 → 回滚游标
  lastAgentTimestamp[chatJid] = previousCursor;
  saveState();
  return false;  // → GroupQueue 触发重试
}
```

**为什么区分"有输出"和"没输出"？**

| 容器是否发过消息 | 是否回滚游标 | 原因 |
|----------------|------------|------|
| 没发过 | ✅ 回滚 | 消息完全没被处理，需要重试 |
| 发过 | ❌ 不回滚 | 用户已收到回复，回滚会导致重复回复 |

## 场景 5：指数退避重试

当容器多次失败时，重试间隔会指数增长：

```typescript
// group-queue.ts → scheduleRetry()

const MAX_RETRIES = 5;
const BASE_RETRY_MS = 5000;

private scheduleRetry(groupJid: string, state: GroupState): void {
  state.retryCount++;

  if (state.retryCount > MAX_RETRIES) {
    // 5 次失败后放弃（等待下一条新消息触发）
    state.retryCount = 0;
    return;
  }

  // 指数退避: 5s → 10s → 20s → 40s → 80s
  const delayMs = BASE_RETRY_MS * Math.pow(2, state.retryCount - 1);
  setTimeout(() => {
    this.enqueueMessageCheck(groupJid);
  }, delayMs);
}
```

```
重试 1: 等待 5 秒
重试 2: 等待 10 秒
重试 3: 等待 20 秒
重试 4: 等待 40 秒
重试 5: 等待 80 秒
放弃 (等下一条新消息到来后才会重新尝试)
```

## 场景 6：定时任务与用户消息的竞争

一个群组同时有用户消息待处理和定时任务到期：

```
状态:
  family-chat: 有 2 条未处理消息 + 1 个到期的定时任务
  容器槽位: 有空闲

处理:
```

```typescript
// group-queue.ts → drainGroup()

private drainGroup(groupJid: string): void {
  const state = this.getGroup(groupJid);

  // 优先级 1: 定时任务（它们不会被重新发现）
  if (state.pendingTasks.length > 0) {
    const task = state.pendingTasks.shift()!;
    this.runTask(groupJid, task);
    return;  // 任务完成后 drainGroup 会再次被调用
  }

  // 优先级 2: 用户消息（它们保存在 SQLite 中，可以重新发现）
  if (state.pendingMessages) {
    this.runForGroup(groupJid, 'drain');
    return;
  }

  // 都没有了 → 检查其他群组
  this.drainWaiting();
}
```

**任务优先于消息**的原因：任务一旦错过执行窗口（如 cron 表达式 `0 9 * * *` 的 9:00 窗口），可能就不适合再运行了。而消息保存在 SQLite 中，随时可以重新读取。

同一群组的任务和消息是**串行的**——一个容器处理完，释放后才处理下一个。这避免了同一群组两个容器同时操作相同文件的冲突。

## 场景 7：宿主机进程重启

NanoClaw 进程崩溃或被重启后：

```typescript
// index.ts → recoverPendingMessages()

function recoverPendingMessages(): void {
  for (const [chatJid, group] of Object.entries(registeredGroups)) {
    const sinceTimestamp = lastAgentTimestamp[chatJid] || '';
    const pending = getMessagesSince(chatJid, sinceTimestamp, ASSISTANT_NAME);
    if (pending.length > 0) {
      // 找到崩溃前没处理完的消息
      queue.enqueueMessageCheck(chatJid);
    }
  }
}
```

### 容器在宿主机重启后的命运

```typescript
// group-queue.ts → shutdown()

async shutdown(): Promise<void> {
  this.shuttingDown = true;

  // 注意: 不杀死活跃容器！
  // 它们会通过 idle timeout 或 container timeout 自行退出
  // --rm 标志确保退出后自动清理
}
```

```
宿主机重启前:
  容器 A: 正在工作

宿主机重启后:
  容器 A: 仍在运行（它是独立的 Docker 进程）
  新的宿主机进程: 启动，不知道容器 A 的存在

容器 A 最终:
  → 空闲超时 → _close 哨兵被写入（但不是由新宿主机写的）
  → 实际上 _close 不会被写，因为新宿主机不知道这个容器
  → 硬超时到达 → Docker 停止容器 → --rm 自动清理

同时:
  新宿主机进程启动 → 回收挂起消息 → 启动新容器处理
  → 用户可能收到重复回复（如果旧容器发过消息的话）
```

**这是一个已知的边界情况**——宿主机重启时正好有活跃容器，可能导致重复回复。NanoClaw 选择"宁可重复也不丢失"的策略。

## 场景 8：IPC 文件处理失败

```typescript
// ipc.ts — 错误隔离

try {
  const data = JSON.parse(fs.readFileSync(filePath, 'utf-8'));
  // ... 处理 ...
  fs.unlinkSync(filePath);
} catch (err) {
  // 处理失败 → 移到 errors/ 目录（不阻塞其他文件）
  const errorDir = path.join(ipcBaseDir, 'errors');
  fs.mkdirSync(errorDir, { recursive: true });
  fs.renameSync(filePath, path.join(errorDir, `${sourceGroup}-${file}`));
}
```

**设计原则：一个损坏的 IPC 文件不应该影响其他群组或其他 IPC 操作。**

## 场景 9：跨群组授权拦截

一个非 main 群组的 Agent 试图给另一个群组发消息或创建任务：

```
容器 (family-chat) 调用:
  mcp__nanoclaw__send_message({ text: "hacked!", chatJid: "work-team-jid" })

MCP Server 写入 IPC 文件:
  data/ipc/family-chat/messages/xxx.json
  → chatJid: work-team-jid（目标群组）

宿主机 IPC Watcher 读取:
  sourceGroup = "family-chat" (从目录路径确定)
  targetGroup = registeredGroups["work-team-jid"] → folder = "work-team"

授权检查:
  isMain = false
  targetGroup.folder ("work-team") !== sourceGroup ("family-chat")
  → ❌ 拒绝！

  logger.warn('Unauthorized IPC message attempt blocked');
  fs.unlinkSync(filePath);  // 删除 IPC 文件，不发送
```

## 场景 10：同一群组任务完成后排水

```
family-chat 有以下待处理项:
  - 定时任务 A（到期）
  - 定时任务 B（到期）
  - 3 条用户消息

处理顺序:
1. 启动容器处理任务 A
2. 任务 A 完成 → drainGroup()
3. 发现任务 B → 启动容器处理任务 B
4. 任务 B 完成 → drainGroup()
5. 发现用户消息 → 启动容器处理消息
6. 消息处理完成 → drainGroup()
7. 无待处理项 → drainWaiting() 检查其他群组
```

注意每次都是**串行**的——同一群组不会同时有两个容器运行。

## 总结：设计决策背后的思考

| 边界情况 | 策略 | 代价 |
|---------|------|------|
| 并发满了 | 排队，释放后处理 | 用户等待时间增加 |
| 容器崩溃 | 回滚游标 + 指数退避 | 最多等 ~160 秒 |
| 已发消息但容器崩溃 | 不回滚 | 可能丢失后续内容 |
| 宿主机重启 | 恢复挂起消息 | 可能重复回复 |
| IPC 文件损坏 | 移到 errors/ | 一次操作丢失 |
| 跨群组攻击 | 目录身份 + 授权检查 | 无 |
| 任务 vs 消息 | 任务优先 | 消息稍微延迟 |

NanoClaw 的设计哲学是：**简单优于复杂，宁可略有重复也不丢失**。没有分布式锁、没有消息队列中间件、没有复杂的共识协议——只有文件、SQLite 和清晰的状态管理。
