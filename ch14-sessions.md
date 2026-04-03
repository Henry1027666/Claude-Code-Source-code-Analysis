# 第 14 章：会话管理

## 14.1 会话生命周期

Claude Code 的会话管理涵盖了从创建到恢复的完整生命周期：

```
会话创建
    │
    ├─ 生成唯一 SessionId（UUID）
    ├─ 初始化消息历史
    ├─ 注册并发会话
    └─ 启动 REPL 或无头模式
    │
    ▼
会话运行
    │
    ├─ 处理用户消息
    ├─ 执行工具调用
    ├─ 持久化对话记录
    └─ 更新会话标题
    │
    ▼
会话结束
    │
    ├─ 刷新会话存储
    ├─ 清理临时文件
    └─ 注销并发会话
```

---

## 14.2 会话持久化

[src/utils/sessionStorage.ts](../../src/utils/sessionStorage.ts) 处理会话的持久化：

```typescript
// 会话存储路径
// ~/.claude/projects/<sanitized-project-path>/sessions/<sessionId>.jsonl

export async function recordTranscript(
  sessionId: SessionId,
  message: Message,
): Promise<void> {
  const path = getSessionTranscriptPath(sessionId);
  
  // 追加写入（JSONL 格式，每行一条消息）
  await fs.appendFile(path, JSON.stringify(message) + '\n');
}

export async function flushSessionStorage(sessionId: SessionId): Promise<void> {
  // 确保所有消息都已写入磁盘
  await fsync(sessionId);
}
```

### 会话标题缓存

```typescript
export async function cacheSessionTitle(
  sessionId: SessionId,
  title: string,
): Promise<void> {
  // 保存会话标题，用于 /resume 命令显示
  await saveToSessionIndex(sessionId, { title, updatedAt: Date.now() });
}
```

---

## 14.3 会话恢复

用户可以通过 `/resume` 命令恢复之前的会话：

```typescript
// src/utils/conversationRecovery.ts
export async function loadConversationForResume(
  sessionId: SessionId,
): Promise<Message[]> {
  const transcriptPath = getSessionTranscriptPath(sessionId);
  
  // 读取 JSONL 格式的对话记录
  const lines = await fs.readFile(transcriptPath, 'utf8');
  return lines
    .split('\n')
    .filter(Boolean)
    .map(line => JSON.parse(line));
}
```

---

## 14.4 并发会话管理

[src/utils/concurrentSessions.ts](../../src/utils/concurrentSessions.ts) 追踪同时运行的会话：

```typescript
export async function countConcurrentSessions(): Promise<number> {
  // 检查有多少个 Claude Code 实例在运行
  const sessions = await listActiveSessions();
  return sessions.length;
}

export async function registerSession(sessionId: SessionId): Promise<void> {
  // 在共享文件中注册当前会话
  await addToSessionRegistry(sessionId);
}
```

---

## 14.5 远程会话

[src/remote/RemoteSessionManager.ts](../../src/remote/RemoteSessionManager.ts) 管理远程会话：

```typescript
export function createRemoteSessionConfig(
  sessionUrl: string,
  authToken: string,
): RemoteSessionConfig {
  return {
    url: sessionUrl,
    token: authToken,
    // WebSocket 连接配置
    reconnectInterval: 5000,
    maxReconnectAttempts: 10,
  };
}
```

---

## 14.6 Worktree 隔离

[src/tools/EnterWorktreeTool/](../../src/tools/EnterWorktreeTool/) 实现了 Git Worktree 隔离：

```typescript
// 创建隔离的 Git Worktree
export async function enterWorktree(
  name: string,
  context: ToolUseContext,
): Promise<WorktreeResult> {
  const worktreePath = path.join('.claude', 'worktrees', name);
  
  // 创建新的 Git Worktree
  await execa('git', ['worktree', 'add', worktreePath, '-b', `claude/${name}`]);
  
  // 切换工作目录
  setCwd(worktreePath);
  
  return {
    path: worktreePath,
    branch: `claude/${name}`,
  };
}
```

---

## 小结

Claude Code 的会话管理系统提供了完整的会话生命周期支持：
- JSONL 格式的持久化确保了对话记录的可靠性
- 会话恢复功能让用户可以继续之前的工作
- Worktree 隔离支持并行开发不同功能
- 远程会话支持跨设备协作
