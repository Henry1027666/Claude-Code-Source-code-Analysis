# 第 3 章：查询引擎与 Agent 循环

## 3.1 架构概述

查询引擎是 Claude Code 的核心——它驱动着 AI 与工具之间的交互循环。整个系统由两个关键文件构成：

- [src/QueryEngine.ts](../../src/QueryEngine.ts)：高层会话管理，封装多轮对话状态
- [src/query.ts](../../src/query.ts)：底层查询循环，实现单次 AI 调用的完整生命周期

```
用户消息
    │
    ▼
QueryEngine.submitMessage()   ← 会话级别（跨轮次持久化状态）
    │
    ▼
query()                        ← 单次查询入口
    │
    ▼
queryLoop()                    ← 核心循环（while(true)）
    │
    ├─► callModel()            ← 调用 Claude API（流式）
    ├─► StreamingToolExecutor  ← 并发工具执行
    ├─► checkTokenBudget()     ← Token 预算检查
    └─► 错误恢复逻辑
```

---

## 3.2 QueryEngine：会话状态管理

[src/QueryEngine.ts:184](../../src/QueryEngine.ts#L184) 定义了 `QueryEngine` 类：

```typescript
export class QueryEngine {
  private config: QueryEngineConfig
  private mutableMessages: Message[]        // 对话历史（跨轮次）
  private abortController: AbortController  // 取消控制
  private permissionDenials: SDKPermissionDenial[]  // 权限拒绝记录
  private totalUsage: NonNullableUsage      // 累计 Token 使用量
  private readFileState: FileStateCache     // 文件状态缓存
  private discoveredSkillNames = new Set<string>()  // 本轮发现的技能
  private loadedNestedMemoryPaths = new Set<string>()  // 已加载的内存路径
}
```

### QueryEngineConfig 配置项

```typescript
type QueryEngineConfig = {
  cwd: string                    // 工作目录
  tools: Tools                   // 可用工具列表
  commands: Command[]            // 可用命令列表
  mcpClients: MCPServerConnection[]  // MCP 服务器连接
  agents: AgentDefinition[]      // Agent 定义
  canUseTool: CanUseToolFn       // 权限检查函数
  getAppState: () => AppState    // 读取 UI 状态
  setAppState: (f) => void       // 更新 UI 状态
  initialMessages?: Message[]    // 初始消息（用于恢复会话）
  readFileCache: FileStateCache  // 文件读取缓存
  customSystemPrompt?: string    // 自定义系统提示
  appendSystemPrompt?: string    // 追加系统提示
  userSpecifiedModel?: string    // 用户指定模型
  maxTurns?: number              // 最大轮次限制
  maxBudgetUsd?: number          // 最大费用限制
  taskBudget?: { total: number } // 任务 Token 预算
  thinkingConfig?: ThinkingConfig  // 思考模式配置
  // ...
}
```

### submitMessage：单次用户消息处理

`submitMessage()` 是一个异步生成器（AsyncGenerator），它流式产出 `SDKMessage` 事件：

```typescript
async *submitMessage(
  prompt: string | ContentBlockParam[],
  options?: { uuid?: string; isMeta?: boolean },
): AsyncGenerator<SDKMessage, void, unknown> {
  // 1. 清理上轮状态
  this.discoveredSkillNames.clear();
  
  // 2. 包装权限检查（追踪拒绝记录）
  const wrappedCanUseTool: CanUseToolFn = async (...) => {
    const result = await canUseTool(...);
    if (result.behavior !== 'allow') {
      this.permissionDenials.push({ tool_name, tool_use_id, tool_input });
    }
    return result;
  };
  
  // 3. 构建系统提示
  const { defaultSystemPrompt, userContext, systemContext } = 
    await fetchSystemPromptParts({ tools, mainLoopModel, ... });
  
  // 4. 注入内存提示（如果有自定义内存路径）
  const memoryMechanicsPrompt = customPrompt && hasAutoMemPathOverride()
    ? await loadMemoryPrompt()
    : null;
  
  // 5. 委托给 query() 执行
  yield* query({ messages, systemPrompt, canUseTool, ... });
}
```

---

## 3.3 queryLoop：核心循环机制

[src/query.ts:241](../../src/query.ts#L241) 中的 `queryLoop()` 是整个系统最核心的函数。它是一个 `while(true)` 循环，每次迭代代表一次 AI 调用：

```typescript
async function* queryLoop(params: QueryParams, consumedCommandUuids: string[]) {
  // 初始化跨迭代状态
  let state: State = {
    messages: params.messages,
    toolUseContext: params.toolUseContext,
    maxOutputTokensRecoveryCount: 0,
    hasAttemptedReactiveCompact: false,
    turnCount: 1,
    transition: undefined,  // 上次迭代继续的原因
    // ...
  };
  
  const budgetTracker = createBudgetTracker();
  
  while (true) {
    // 解构当前状态
    const { messages, toolUseContext, ... } = state;
    
    // 1. 预处理：应用工具结果预算、历史压缩
    messagesForQuery = await applyToolResultBudget(messagesForQuery, ...);
    
    // 2. 调用 Claude API（流式）
    yield { type: 'stream_request_start' };
    const response = yield* callModel(messagesForQuery, systemPrompt, ...);
    
    // 3. 处理工具调用
    const toolResults = yield* executeTools(response.toolCalls, ...);
    
    // 4. 决定是否继续循环
    const decision = checkContinuation(response, toolResults, budgetTracker);
    
    if (decision.type === 'terminal') {
      return decision;  // 退出循环
    }
    
    // 5. 更新状态，继续下一轮
    state = { ...state, messages: [...messages, ...newMessages], transition: decision };
  }
}
```

### 循环状态机

循环的每次迭代都有明确的"继续原因"（`transition`），这使得调试和测试变得清晰：

```typescript
type Continue = 
  | { type: 'tool_use' }                    // AI 调用了工具
  | { type: 'token_budget_continuation' }   // Token 预算未用完，继续
  | { type: 'max_output_tokens_recovery' }  // 输出被截断，恢复
  | { type: 'prompt_too_long_compact' }     // 上下文太长，压缩后重试
  | { type: 'fallback_model' }              // 切换到备用模型
  | { type: 'stop_hook_continuation' }      // Stop Hook 要求继续

type Terminal = 
  | { type: 'end_turn' }                    // 正常结束
  | { type: 'max_turns' }                   // 达到最大轮次
  | { type: 'max_budget' }                  // 超出费用预算
  | { type: 'error' }                       // 不可恢复错误
```

---

## 3.4 流式工具执行

### StreamingToolExecutor

[src/services/tools/StreamingToolExecutor.ts](../../src/services/tools/StreamingToolExecutor.ts) 实现了流式工具执行的核心逻辑：

**关键设计**：工具调用在 AI 流式输出过程中**实时发现、实时开始执行**，不等待 AI 完成整个响应。

```
AI 流式输出:
  "我来读取文件..."  → 显示给用户
  [tool_use: FileRead] → 立即加入执行队列！
  "...然后分析内容"  → 继续显示
  [tool_use: GrepTool] → 立即加入执行队列！
  
工具执行（与 AI 输出并行）:
  FileRead 开始执行 ──────────────────► 完成
  GrepTool 开始执行（如果是只读工具）──► 完成
```

### 并发规则

```typescript
canExecuteTool(isConcurrencySafe: boolean): boolean {
  const running = tools.filter(t => t.status === 'executing');
  
  // 规则：
  // 1. 没有工具在运行 → 可以执行
  // 2. 所有运行中的工具都是"并发安全"的，且新工具也是 → 可以并行
  // 3. 否则 → 等待
  return running.length === 0 ||
    (isConcurrencySafe && running.every(t => t.isConcurrencySafe));
}
```

**并发安全工具**（只读，可并行）：
- `FileReadTool`、`GlobTool`、`GrepTool`
- `WebFetchTool`、`WebSearchTool`

**非并发安全工具**（写入，必须串行）：
- `FileWriteTool`、`FileEditTool`
- `BashTool`、`PowerShellTool`

---

## 3.5 Token 预算管理

[src/query/tokenBudget.ts](../../src/query/tokenBudget.ts) 实现了防止 AI 无限循环的 Token 预算系统：

```typescript
const COMPLETION_THRESHOLD = 0.9;   // 90% 使用率触发停止
const DIMINISHING_THRESHOLD = 500;  // 连续增量 < 500 token 视为"原地打转"

export function checkTokenBudget(
  tracker: BudgetTracker,
  agentId: string | undefined,
  budget: number | null,
  globalTurnTokens: number,
): TokenBudgetDecision {
  // 子 Agent 不受预算限制（由父 Agent 控制）
  if (agentId || budget === null || budget <= 0) {
    return { action: 'stop', completionEvent: null };
  }
  
  const pct = Math.round((turnTokens / budget) * 100);
  const deltaSinceLastCheck = globalTurnTokens - tracker.lastGlobalTurnTokens;
  
  // 检测"原地打转"：连续 3 次续写，每次新增 < 500 token
  const isDiminishing =
    tracker.continuationCount >= 3 &&
    deltaSinceLastCheck < DIMINISHING_THRESHOLD &&
    tracker.lastDeltaTokens < DIMINISHING_THRESHOLD;
  
  // 未达到 90% 且没有原地打转 → 继续
  if (!isDiminishing && turnTokens < budget * COMPLETION_THRESHOLD) {
    tracker.continuationCount++;
    return {
      action: 'continue',
      nudgeMessage: getBudgetContinuationMessage(pct, turnTokens, budget),
      // ...
    };
  }
  
  // 停止
  return { action: 'stop', completionEvent: { ... } };
}
```

### 预算决策流程

```
每次 AI 响应后：
    │
    ▼
已使用 Token < 90% 预算？
    │
    ├─ 是 → 连续 3 次且每次增量 < 500？
    │         ├─ 是（原地打转）→ 停止
    │         └─ 否 → 发送"继续"消息，再次调用 AI
    │
    └─ 否（≥ 90%）→ 停止
```

---

## 3.6 错误恢复机制

`queryLoop` 实现了多层错误恢复，而不是遇到错误就崩溃：

### 1. 上下文过长（413 错误）

```
错误：消息历史太长，超过模型上下文窗口
    │
    ▼
触发自动压缩（autoCompact）
    │
    ▼
将历史消息压缩为摘要
    │
    ▼
用压缩后的消息重试
```

### 2. 输出截断（max_output_tokens）

```
错误：AI 输出被截断（达到 max_output_tokens 限制）
    │
    ▼
最多重试 3 次（MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3）
    │
    ▼
发送"请继续"消息，让 AI 接着输出
```

### 3. 模型降级

```
错误：主模型不可用或超出配额
    │
    ▼
切换到 fallbackModel（备用模型）
    │
    ▼
用备用模型重试
```

### 4. 图像处理错误

```typescript
// 图像太大或格式不支持
if (error instanceof ImageSizeError || error instanceof ImageResizeError) {
  // 将图像替换为错误消息，继续执行
  yield createAssistantAPIErrorMessage(error.message);
  // 不中断循环
}
```

---

## 3.7 系统提示构建

每次 `submitMessage()` 调用都会重新构建系统提示：

```typescript
const { defaultSystemPrompt, userContext, systemContext } = 
  await fetchSystemPromptParts({
    tools,
    mainLoopModel,
    additionalWorkingDirectories,
    mcpClients,
    customSystemPrompt,
  });

const systemPrompt = asSystemPrompt([
  // 1. 自定义提示（如果有）或默认提示
  ...(customPrompt !== undefined ? [customPrompt] : defaultSystemPrompt),
  // 2. 内存机制提示（如果启用了自定义内存路径）
  ...(memoryMechanicsPrompt ? [memoryMechanicsPrompt] : []),
  // 3. 追加提示（SDK 调用者可以添加额外指令）
  ...(appendSystemPrompt ? [appendSystemPrompt] : []),
]);
```

系统提示包含：
- Claude 的角色和行为指令
- 可用工具的描述
- 工作目录信息
- 用户上下文（git 状态、项目信息等）
- 内存内容（MEMORY.md）

---

## 3.8 消息历史压缩

当对话历史过长时，系统会自动压缩：

```typescript
// 自动压缩触发条件
const tokenWarning = calculateTokenWarningState(
  contextTokens,
  contextWindow,
  autoCompactTracking,
);

if (tokenWarning === 'compact_needed') {
  // 调用 Claude API 生成历史摘要
  const summary = await compactMessages(messages);
  
  // 用摘要替换历史消息
  const compactedMessages = buildPostCompactMessages(messages, summary);
  
  // 继续循环（使用压缩后的消息）
  state = { ...state, messages: compactedMessages };
  continue;
}
```

---

## 3.9 完整流程图

```
submitMessage("帮我修复 bug")
    │
    ▼
构建系统提示 + 用户上下文
    │
    ▼
┌─────────────────────────────────────────────────────┐
│                   queryLoop()                        │
│                                                      │
│  ┌─────────────────────────────────────────────┐    │
│  │  迭代 1                                      │    │
│  │  ├─ 应用工具结果预算                          │    │
│  │  ├─ 调用 Claude API（流式）                   │    │
│  │  │   ├─ 发现 tool_use: FileRead → 立即执行   │    │
│  │  │   └─ 发现 tool_use: GrepTool → 立即执行   │    │
│  │  ├─ 等待所有工具完成                          │    │
│  │  └─ transition: 'tool_use' → 继续            │    │
│  └─────────────────────────────────────────────┘    │
│                                                      │
│  ┌─────────────────────────────────────────────┐    │
│  │  迭代 2                                      │    │
│  │  ├─ 调用 Claude API（携带工具结果）            │    │
│  │  │   ├─ 发现 tool_use: FileEdit → 执行       │    │
│  │  │   └─ AI 输出文字说明                       │    │
│  │  ├─ 检查 Token 预算（< 90%）→ 继续            │    │
│  │  └─ transition: 'tool_use' → 继续            │    │
│  └─────────────────────────────────────────────┘    │
│                                                      │
│  ┌─────────────────────────────────────────────┐    │
│  │  迭代 3                                      │    │
│  │  ├─ 调用 Claude API                          │    │
│  │  │   └─ 无工具调用，只有文字响应              │    │
│  │  └─ terminal: 'end_turn' → 退出循环          │    │
│  └─────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────┘
    │
    ▼
返回最终响应给用户
```

---

## 小结

查询引擎的设计体现了几个关键工程决策：

1. **流式 + 工具执行重叠**：工具在 AI 还没说完时就开始执行，大幅减少等待时间
2. **显式状态机**：每次循环继续都有明确的 `transition` 原因，便于调试和测试
3. **多层错误恢复**：上下文压缩、输出截断恢复、模型降级，而非直接崩溃
4. **Token 预算防护**：检测"原地打转"行为，防止无限循环消耗资源
5. **并发安全工具执行**：只读工具并行，写入工具串行，平衡性能与安全

下一章，我们将深入工具系统——这 45 个工具是 AI 与真实世界交互的唯一通道。
