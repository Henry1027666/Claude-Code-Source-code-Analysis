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

[src/tools/AgentTool/](../../src/tools/AgentTool/) 实现了子 Agent 系统：

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
];
```

### 子 Agent 执行流程

```typescript
// src/tools/AgentTool/runAgent.ts
export async function runAgent(
  agentDefinition: AgentDefinition,
  prompt: string,
  context: ToolUseContext,
): Promise<AgentResult> {
  // 1. 创建隔离的子 Agent 上下文
  const subagentContext = createSubagentContext(context, agentDefinition);
  
  // 2. 构建子 Agent 系统提示
  const systemPrompt = buildAgentSystemPrompt(agentDefinition);
  
  // 3. 执行独立的查询循环
  const messages: Message[] = [];
  for await (const event of query({
    messages,
    systemPrompt,
    toolUseContext: subagentContext,
    // 子 Agent 有独立的工具集
    tools: agentDefinition.tools,
  })) {
    // 收集子 Agent 的输出
    if (event.type === 'assistant') {
      messages.push(event);
    }
  }
  
  // 4. 保存内存快照（用于后续恢复）
  await saveAgentMemorySnapshot(subagentContext.agentId, messages);
  
  return { result: extractFinalResult(messages) };
}
```

---

## 12.3 子 Agent 上下文隔离

```typescript
// src/tools/AgentTool/forkSubagent.ts
export function createSubagentContext(
  parentContext: ToolUseContext,
  agentDef: AgentDefinition,
): ToolUseContext {
  const agentId = generateAgentId();
  
  return {
    ...parentContext,
    agentId,
    agentType: agentDef.name,
    
    // 子 Agent 的 setAppState 是 no-op
    // 避免子 Agent 修改主线程 UI 状态
    setAppState: (f) => {
      // 只更新任务相关状态（通过 setAppStateForTasks）
    },
    
    // 子 Agent 有独立的文件缓存
    readFileState: cloneFileStateCache(parentContext.readFileState),
    
    // 子 Agent 有独立的内容替换状态
    contentReplacementState: cloneContentReplacementState(
      parentContext.contentReplacementState
    ),
  };
}
```

---

## 12.4 Coordinator 模式

[src/coordinator/coordinatorMode.ts](../../src/coordinator/coordinatorMode.ts) 实现了 Anthropic 内部的多 Agent 协调模式：

```typescript
// Coordinator 模式通过环境变量启用
export function isCoordinatorMode(): boolean {
  if (feature('COORDINATOR_MODE')) {
    return isEnvTruthy(process.env.CLAUDE_CODE_COORDINATOR_MODE);
  }
  return false;
}

// Coordinator 专用工具集
const COORDINATOR_TOOLS = [
  TEAM_CREATE_TOOL_NAME,    // 创建 Worker 团队
  TEAM_DELETE_TOOL_NAME,    // 删除 Worker 团队
  SEND_MESSAGE_TOOL_NAME,   // 向 Worker 发送消息
  TASK_STOP_TOOL_NAME,      // 停止任务
];
```

### Coordinator 工作流

```
Coordinator Agent
    │
    ├─ TeamCreateTool → 创建 Worker 团队
    │
    ├─ SendMessageTool → 向 Worker 1 分配任务 A
    ├─ SendMessageTool → 向 Worker 2 分配任务 B
    ├─ SendMessageTool → 向 Worker 3 分配任务 C
    │
    ├─ 等待所有 Worker 完成
    │
    └─ 汇总结果，生成最终报告
```

---

## 12.5 Agent 颜色管理

[src/tools/AgentTool/agentColorManager.ts](../../src/tools/AgentTool/agentColorManager.ts) 为不同 Agent 分配不同颜色，便于在 UI 中区分：

```typescript
const AGENT_COLORS: AgentColorName[] = [
  'blue', 'green', 'yellow', 'magenta', 'cyan',
  'red', 'white', 'gray',
];

export function assignAgentColor(agentId: string): AgentColorName {
  // 基于 agentId 哈希确定颜色，保证同一 Agent 颜色一致
  const hash = hashString(agentId);
  return AGENT_COLORS[hash % AGENT_COLORS.length];
}
```

---

## 12.6 Agent 内存快照

子 Agent 完成后，会保存内存快照用于后续恢复：

```typescript
// src/tools/AgentTool/agentMemorySnapshot.ts
export async function saveAgentMemorySnapshot(
  agentId: AgentId,
  messages: Message[],
): Promise<void> {
  const snapshotPath = getAgentSnapshotPath(agentId);
  
  await fs.writeFile(snapshotPath, JSON.stringify({
    agentId,
    messages,
    savedAt: Date.now(),
  }));
}

// 恢复子 Agent 会话
export async function resumeAgent(agentId: AgentId): Promise<Message[]> {
  const snapshot = await loadAgentMemorySnapshot(agentId);
  return snapshot.messages;
}
```

---

## 小结

Claude Code 的多 Agent 系统提供了灵活的任务并行化能力：
- **AgentTool** 允许主 Agent 派发子任务给专门的子 Agent
- **内置 Agent 类型** 针对不同场景优化了工具集和系统提示
- **上下文隔离** 确保子 Agent 不会意外修改主线程状态
- **内存快照** 支持子 Agent 会话的恢复和继续
