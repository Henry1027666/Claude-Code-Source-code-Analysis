# 第 10 章：VS Code Bridge

## 10.1 Bridge 架构概述

VS Code Bridge 是 Claude Code 与 IDE 集成的核心机制。它允许用户在 VS Code 中使用 Claude Code，同时保持终端 CLI 的完整功能。

```
VS Code 扩展
    │
    │ WebSocket / HTTP
    ▼
Bridge 服务器（本地机器）
    │
    ├─ bridgeMain.ts    ← 主协调器
    ├─ replBridge.ts    ← REPL 会话管理
    └─ bridgeApi.ts     ← API 客户端
    │
    ▼
Claude Code REPL 实例
```

---

## 10.2 Bridge 启动流程

[src/bridge/bridgeMain.ts](../../src/bridge/bridgeMain.ts) 是 Bridge 模式的主入口：

```typescript
// 通过 claude remote-control 或 claude bridge 启动
if (args[0] === 'remote-control' || args[0] === 'bridge') {
  const { runBridgeMain } = await import('../bridge/bridgeMain.js');
  await runBridgeMain(bridgeConfig);
}
```

Bridge 启动后：
1. 向 Anthropic 服务器注册（获取 Bridge ID）
2. 开始轮询新的会话请求
3. 为每个请求创建独立的 REPL 实例
4. 通过 WebSocket 双向传递消息

---

## 10.3 会话管理

```typescript
// src/bridge/sessionRunner.ts
export function createSessionSpawner(config: BridgeConfig): SessionSpawner {
  return {
    async spawn(opts: SessionSpawnOpts): Promise<SessionHandle> {
      // 为每个 IDE 会话创建独立的 Claude Code 进程
      const session = await spawnClaudeCodeProcess({
        cwd: opts.workingDirectory,
        model: opts.model,
        // ...
      });
      
      return {
        sessionId: session.id,
        send: (msg) => session.stdin.write(JSON.stringify(msg)),
        onMessage: (handler) => session.stdout.on('data', handler),
        kill: () => session.kill(),
      };
    }
  };
}
```

---

## 10.4 消息协议

Bridge 使用 JSON-RPC 风格的消息协议：

```typescript
// IDE → Claude Code
{
  type: 'user_message',
  content: '帮我重构这个函数',
  context: {
    selectedText: '...',
    filePath: '/src/utils.ts',
    cursorPosition: { line: 42, col: 10 },
  }
}

// Claude Code → IDE
{
  type: 'assistant_message',
  content: '我来分析这个函数...',
  toolUse: {
    name: 'FileRead',
    input: { file_path: '/src/utils.ts' }
  }
}
```

---

## 10.5 Worktree 隔离

Bridge 模式支持为每个会话创建独立的 Git Worktree：

```typescript
// src/utils/worktree.ts
export async function createAgentWorktree(
  sessionId: string,
  baseDir: string,
): Promise<string> {
  const worktreePath = path.join(tmpdir(), `claude-worktree-${sessionId}`);
  
  // 创建新的 Git Worktree
  await execa('git', ['worktree', 'add', worktreePath, '-b', `claude/${sessionId}`]);
  
  return worktreePath;
}
```

这确保了不同 IDE 会话之间的文件操作相互隔离。

---

## 10.6 JWT Token 刷新

Bridge 使用 JWT Token 进行认证，需要定期刷新：

```typescript
// src/bridge/jwtUtils.ts
export function createTokenRefreshScheduler(
  getToken: () => Promise<string>,
  onRefresh: (token: string) => void,
): TokenRefreshScheduler {
  // 在 Token 过期前 5 分钟自动刷新
  const REFRESH_BEFORE_EXPIRY_MS = 5 * 60 * 1000;
  
  return {
    start() {
      scheduleRefresh();
    },
    stop() {
      clearTimeout(refreshTimer);
    }
  };
}
```

---

## 小结

VS Code Bridge 是一个复杂的会话管理系统，它将 Claude Code 的终端能力无缝集成到 IDE 环境中，同时保持了会话隔离和安全性。
