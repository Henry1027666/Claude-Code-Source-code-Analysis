# 第 12 章：多 Agent 协调

## 12.1 多 Agent 架构概述

Claude Code 支持多种多 Agent 协作模式：

```
单 Agent 模式（默认）:
  主 Agent → 工具调用 → 结果

子 Agent 模式（AgentTool）:
  主 Agent → AgentTool → 子 Agent 1
                       → 子 Agent 2（并行）
                       → 子 Agent 3（并行）

Coordinator 模式（企业内部）:
  Coordinator Agent → 任务分发 → Worker Agent 1
                               → Worker Agent 2
                               → Worker Agent 3
```

---

## 12.2 AgentTool：子 Agent 派发

[src/tools/AgentTool/](../../src/tools/AgentTool/) 实现了子 Agent 系统。

### 内置 Agent 类型

```typescript
// src/tools/AgentTool/builtInAgents.ts
const BUILT_IN_AGENTS = [
  {
    name: 'general-purpose',
    description: '通用 Agent，适合复杂的多步骤任务',
    tools: ALL_TOOLS,
  },
  {
    name: 'Explore',
    description: '代码库探索专家，快速理解代码结构',
    tools: [FileReadTool, GlobTool, GrepTool, BashTool],
  },
  {
    name: 'Plan',
    description: '实现规划专家，设计技术方案',
    tools: [FileReadTool, GlobTool, GrepTool],
  },
  {
    name: 'claude-code-guide',
    description: 'Claude Code 使用指南专家',
    tools: [WebFetchTool, WebSearchTool],
  },
]
```

### 子 Agent 的完整执行流程

AgentTool 的 `call()` 方法处理多种执行路径：

```
AgentTool.call(input, context)
    │
    ├─ [isolation: 'worktree'] → EnterWorktree → 创建 git worktree 隔离环境
    │
    ├─ [run_in_background: true] → 立即返回 async_launched，后台执行
    │   └─ 超过 120s 自动后台化（assistant 模式）
    │
    ├─ [team_name] → 创建 Teammate（企业多 Agent）
    │
    └─ [默认] → LocalAgentTask
        │
        ├─ 创建隔离的 ToolUseContext（forkSubagent）
        ├─ 调用 query()（独立的 queryLoop）
        ├─ 流式 yield 进度（AgentToolProgress）
        └─ 保存内存快照（agentMemorySnapshot）
```

**后台任务的自动化**：

```typescript
// 超过 120s 的同步子 Agent 自动转为后台任务
const AUTO_BACKGROUND_TIMEOUT_MS = 120_000

// 后台任务的输出文件路径（调用方可以轮询）
const outputFile = getAgentOutputFilePath(agentId)
```

### AgentTool 的输出类型

```typescript
// 同步完成
{ status: 'completed', prompt, result, usage, ... }

// 异步后台启动
{
  status: 'async_launched',
  agentId: string,
  description: string,
  prompt: string,
  outputFile: string,       // 检查进度的文件路径
  canReadOutputFile: boolean,  // 调用方是否有 Read/Bash 工具
}
```

---

## 12.3 子 Agent 上下文隔离

子 Agent 通过 `forkSubagent` 创建完全隔离的执行环境：

```typescript
// src/tools/AgentTool/forkSubagent.ts
export function createSubagentContext(
  parentContext: ToolUseContext,
  agentDef: AgentDefinition,
): ToolUseContext {
  const agentId = generateAgentId()

  return {
    ...parentContext,
    agentId,
    agentType: agentDef.name,

    // 关键隔离点 1：setAppState 替换为 no-op
    // 防止子 Agent 修改主线程 UI 状态（如 REPL 的 spinner、消息列表）
    setAppState: (_f) => {
      // 故意空实现
    },

    // 关键隔离点 2：独立的文件读取缓存
    // 子 Agent 的文件读取不影响父 Agent 的缓存状态
    readFileState: cloneFileStateCache(parentContext.readFileState),

    // 关键隔离点 3：独立的内容替换状态
    // 子 Agent 的工具结果大小控制独立运作
    contentReplacementState: cloneContentReplacementState(
      parentContext.contentReplacementState,
    ),

    // 关键隔离点 4：独立的 queryTracking
    // 子 Agent 的查询链 ID 和深度独立计算
    queryTracking: undefined,
  }
}
```

**`setAppStateForTasks` 的特殊处理**：

虽然 `setAppState` 是 no-op，但 `setAppStateForTasks` 仍然有效——它用于更新任务基础设施状态（如后台任务列表），这些状态需要跨 Agent 共享。

### 子 Agent 的工具集过滤

AgentTool 的 `prompt()` 方法在生成系统提示时会过滤不可用的 Agent：

```typescript
// 过滤 MCP 需求未满足的 Agent
const agentsWithMcpRequirementsMet = filterAgentsByMcpRequirements(agents, mcpServersWithTools)

// 过滤被权限规则拒绝的 Agent
const filteredAgents = filterDeniedAgents(
  agentsWithMcpRequirementsMet,
  toolPermissionContext,
  AGENT_TOOL_NAME,
)
```

这确保 AI 只会看到当前环境中实际可用的 Agent 类型。

---

## 12.4 Coordinator 模式

[src/coordinator/](../../src/coordinator/) 实现了 Anthropic 内部的多 Agent 协调模式，通过环境变量启用：

```typescript
// Coordinator 模式通过 feature gate + 环境变量双重控制
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE)
  }
  return false
}
```

**Coordinator 专用工具集**：

```typescript
const COORDINATOR_TOOLS = [
  TEAM_CREATE_TOOL_NAME,    // 创建 Worker 团队
  TEAM_DELETE_TOOL_NAME,    // 删除 Worker 团队
  SEND_MESSAGE_TOOL_NAME,   // 向 Worker 发送消息
  TASK_STOP_TOOL_NAME,      // 停止任务
]
```

**Coordinator 的 userContext 扩展**：

```typescript
// src/QueryEngine.ts 中的 Coordinator 上下文注入
const userContext = {
  ...baseUserContext,
  ...getCoordinatorUserContext(
    mcpClients,
    isScratchpadEnabled() ? getScratchpadDir() : undefined,
  ),
}
```

### Coordinator 工作流

```
Coordinator Agent
    │
    ├─ TeamCreateTool → 创建 Worker 团队（分配工作目录）
    │
    ├─ SendMessageTool → 向 Worker 1 分配任务 A
    ├─ SendMessageTool → 向 Worker 2 分配任务 B（并行）
    ├─ SendMessageTool → 向 Worker 3 分配任务 C（并行）
    │
    ├─ TaskOutputTool → 轮询各 Worker 的输出
    │
    └─ 汇总结果，生成最终报告
```

## 12.5 Agent 颜色管理

[src/tools/AgentTool/agentColorManager.ts](../../src/tools/AgentTool/agentColorManager.ts) 为不同 Agent 分配不同颜色，便于在 UI 中区分：

```typescript
const AGENT_COLORS: AgentColorName[] = [
  'blue', 'green', 'yellow', 'magenta', 'cyan', 'red', 'white', 'gray',
]

export function assignAgentColor(agentId: string): AgentColorName {
  // 基于 agentId 哈希确定颜色，保证同一 Agent 颜色一致
  const hash = hashString(agentId)
  return AGENT_COLORS[hash % AGENT_COLORS.length]
}
```

## 12.6 Agent 内存快照

子 Agent 完成后，会保存内存快照用于后续恢复：

```typescript
// src/tools/AgentTool/agentMemorySnapshot.ts
export async function saveAgentMemorySnapshot(
  agentId: AgentId,
  messages: Message[],
): Promise<void> {
  const snapshotPath = getAgentSnapshotPath(agentId)
  await fs.writeFile(snapshotPath, JSON.stringify({
    agentId,
    messages,
    savedAt: Date.now(),
  }))
}

// 恢复子 Agent 会话（用于 /resume 命令）
export async function resumeAgent(agentId: AgentId): Promise<Message[]> {
  const snapshot = await loadAgentMemorySnapshot(agentId)
  return snapshot.messages
}
```

**快照的持久化路径**：快照存储在 session 目录下，与主线程的会话文件并列，支持跨进程恢复。

## 12.7 并行子 Agent 的实际执行

当主 Agent 在同一轮次调用多个 AgentTool 时，`StreamingToolExecutor` 的并发规则决定执行策略：

```typescript
// AgentTool 的并发安全性
isConcurrencySafe(input): boolean {
  // 后台 Agent 可以并行（不阻塞主线程）
  if (input.run_in_background) return true
  // 同步 Agent 串行执行（避免资源竞争）
  return false
}
```

因此，多个 `run_in_background: true` 的子 Agent 会真正并行执行，而同步子 Agent 会串行执行。

---

## 小结

Claude Code 的多 Agent 系统提供了灵活的任务并行化能力：
- **AgentTool** 允许主 Agent 派发子任务给专门的子 Agent
- **内置 Agent 类型** 针对不同场景优化了工具集和系统提示
- **上下文隔离** 确保子 Agent 不会意外修改主线程状态
- **内存快照** 支持子 Agent 会话的恢复和继续
