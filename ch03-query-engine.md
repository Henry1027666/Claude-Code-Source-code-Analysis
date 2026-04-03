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

[src/query.ts:241](../../src/query.ts#L241) 中的 `queryLoop()` 是整个系统最核心的函数。它是一个 `while(true)` 循环，每次迭代代表一次 AI 调用。

### 跨迭代状态（State 对象）

循环通过一个不可变的 `State` 对象在迭代间传递状态，每次 `continue` 都创建新的 State 快照：

```typescript
type State = {
  messages: Message[]                          // 当前消息历史
  toolUseContext: ToolUseContext               // 工具执行上下文
  autoCompactTracking: AutoCompactTrackingState | undefined
  maxOutputTokensRecoveryCount: number         // 输出截断恢复次数（上限 3）
  hasAttemptedReactiveCompact: boolean         // 是否已尝试响应式压缩
  maxOutputTokensOverride: number | undefined  // 升级后的 max_tokens（64k）
  pendingToolUseSummary: Promise<...> | undefined
  stopHookActive: boolean | undefined
  turnCount: number
  transition: Continue | undefined             // 上次迭代继续的原因
}
```

### 每次迭代的执行顺序

```
while (true) {
  1. 解构 state（只读快照）
  2. 启动技能发现预取（skillPrefetch，与 API 调用并行）
  3. yield { type: 'stream_request_start' }
  4. 应用工具结果预算（applyToolResultBudget）
  5. 应用 snip 压缩（feature('HISTORY_SNIP')）
  6. 应用 microcompact（缓存感知的轻量压缩）
  7. 应用 context collapse（上下文折叠）
  8. 检查 autocompact 阈值（可能触发完整压缩）
  9. 调用 Claude API（流式，queryModelWithStreaming）
 10. 处理流式响应（工具调用实时发现 → StreamingToolExecutor）
 11. 等待所有工具执行完成（runTools）
 12. 错误恢复判断（prompt_too_long / max_output_tokens / model_fallback）
 13. 执行 stop hooks
 14. 决定继续或退出
}
```

### 循环状态机（transition 类型）

每次 `continue` 都携带明确的原因，便于调试和遥测：

```typescript
// src/query/transitions.ts
type Continue =
  | { reason: 'tool_use' }
  | { reason: 'token_budget_continuation' }
  | { reason: 'max_output_tokens_recovery'; attempt: number }
  | { reason: 'max_output_tokens_escalate' }      // 升级到 64k max_tokens
  | { reason: 'collapse_drain_retry'; committed: number }
  | { reason: 'reactive_compact_retry' }
  | { reason: 'stop_hook_continuation' }
  | { reason: 'stop_hook_blocking_error' }

type Terminal =
  | { reason: 'completed' }
  | { reason: 'aborted_streaming' }
  | { reason: 'prompt_too_long' }
  | { reason: 'image_error' }
  | { reason: 'model_error'; error: unknown }
  | { reason: 'stop_hook_prevented' }
```

### 关键实现细节：工具结果预算

在每次 API 调用前，系统会检查历史消息中工具结果的总大小：

```typescript
// 对超过 maxResultSizeChars 的工具结果进行内容替换
// 无预算限制的工具（如 FileReadTool）被排除在外
messagesForQuery = await applyToolResultBudget(
  messagesForQuery,
  toolUseContext.contentReplacementState,
  persistReplacements ? records => void recordContentReplacement(...) : undefined,
  new Set(
    toolUseContext.options.tools
      .filter(t => !Number.isFinite(t.maxResultSizeChars))
      .map(t => t.name),
  ),
)
```

被替换的内容会写入磁盘，AI 可以通过 FileReadTool 读取完整内容。

---

## 3.4 流式工具执行

### StreamingToolExecutor

[src/services/tools/StreamingToolExecutor.ts](../../src/services/tools/StreamingToolExecutor.ts) 实现了流式工具执行的核心逻辑。

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

### StreamingToolExecutor 的关键方法

```typescript
class StreamingToolExecutor {
  // 当 AI 流中发现新的 tool_use block 时调用
  addTool(toolBlock: ToolUseBlock, parentMessage: AssistantMessage): void
  
  // 同步获取已完成的工具结果（在流式循环中轮询）
  getCompletedResults(): ToolExecutionResult[]
  
  // 异步等待所有剩余工具完成（流结束后调用）
  async *getRemainingResults(): AsyncGenerator<ToolExecutionResult>
  
  // 丢弃所有待执行/执行中的工具（模型降级时使用）
  discard(): void
}
```

**Bash 工具错误传播**：当 Bash 工具执行失败时，会通过 `siblingAbortController` 取消同批次的其他工具，防止在错误状态下继续执行。

### 并发规则

```typescript
// 并发决策逻辑（简化）
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

### 工具执行与流式输出的交织

在 `queryLoop` 中，工具结果的收集与 AI 流式输出交织进行：

```typescript
// 在流式循环内：每次收到 assistant message 就检查是否有工具调用
if (message.type === 'assistant') {
  const toolUseBlocks = message.message.content.filter(
    content => content.type === 'tool_use',
  ) as ToolUseBlock[]
  
  if (streamingToolExecutor) {
    for (const toolBlock of toolUseBlocks) {
      streamingToolExecutor.addTool(toolBlock, message)  // 立即开始执行
    }
  }
}

// 同时轮询已完成的工具结果（不等待流结束）
if (streamingToolExecutor) {
  for (const result of streamingToolExecutor.getCompletedResults()) {
    if (result.message) {
      yield result.message  // 立即 yield 给调用方
      toolResults.push(...)
    }
  }
}
```

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

`queryLoop` 实现了多层错误恢复，而不是遇到错误就崩溃。所有错误恢复都通过创建新的 `State` 对象并 `continue` 来实现，保持循环的不变性。

### 1. 输出截断（max_output_tokens）—— 两阶段恢复

```typescript
// 阶段一：升级 max_tokens（仅触发一次）
// 如果使用了默认的 8k 上限并触发截断，先尝试升级到 64k
if (capEnabled && maxOutputTokensOverride === undefined) {
  state = { ...state, maxOutputTokensOverride: ESCALATED_MAX_TOKENS,
            transition: { reason: 'max_output_tokens_escalate' } }
  continue
}

// 阶段二：多轮恢复（最多 3 次）
// MAX_OUTPUT_TOKENS_RECOVERY_LIMIT = 3
if (maxOutputTokensRecoveryCount < MAX_OUTPUT_TOKENS_RECOVERY_LIMIT) {
  const recoveryMessage = createUserMessage({
    content: `Output token limit hit. Resume directly — no apology, no recap of what you were doing. ` +
             `Pick up mid-thought if that is where the cut happened. Break remaining work into smaller pieces.`,
    isMeta: true,
  })
  state = {
    messages: [...messagesForQuery, ...assistantMessages, recoveryMessage],
    maxOutputTokensRecoveryCount: maxOutputTokensRecoveryCount + 1,
    transition: { reason: 'max_output_tokens_recovery', attempt: maxOutputTokensRecoveryCount + 1 },
    // ...
  }
  continue
}
// 恢复次数耗尽 → 将截断错误 yield 给用户
yield lastMessage
```

### 2. 上下文过长（prompt_too_long / 413 错误）—— 三阶段恢复

```
阶段一：Context Collapse 排空（feature('CONTEXT_COLLAPSE')）
  → 将已折叠的上下文提交，释放 token 空间
  → transition: { reason: 'collapse_drain_retry', committed: N }

阶段二：Reactive Compact（响应式压缩）
  → 调用 Claude API 生成历史摘要
  → 用摘要替换历史消息
  → transition: { reason: 'reactive_compact_retry' }
  → hasAttemptedReactiveCompact = true（防止无限循环）

阶段三：放弃恢复
  → yield 错误消息给用户
  → 执行 stop failure hooks
  → return { reason: 'prompt_too_long' }
```

**关键保护**：`hasAttemptedReactiveCompact` 标志防止压缩后仍然 413 时的无限循环。

### 3. 模型降级（FallbackTriggeredError）

```typescript
if (innerError instanceof FallbackTriggeredError) {
  const fallbackModel = innerError.fallbackModel
  
  // 清空当前迭代的所有结果（防止孤立的 tool_result）
  assistantMessages.length = 0
  toolResults.length = 0
  toolUseBlocks.length = 0
  needsFollowUp = false
  
  // 重建 StreamingToolExecutor（丢弃旧的待执行工具）
  streamingToolExecutor?.discard()
  streamingToolExecutor = new StreamingToolExecutor(...)
  
  // 切换模型
  toolUseContext.options.mainLoopModel = fallbackModel
  
  // 清理 thinking 签名块（模型绑定，不能跨模型复用）
  if (process.env.USER_TYPE === 'ant') {
    messagesForQuery = stripSignatureBlocks(messagesForQuery)
  }
  
  // 通知用户
  yield createSystemMessage(
    `Switched to ${renderModelName(fallbackModel)} due to high demand`,
    'warning',
  )
  continue
}
```

### 4. 图像处理错误

```typescript
if (error instanceof ImageSizeError || error instanceof ImageResizeError) {
  yield createAssistantAPIErrorMessage({ content: error.message })
  return { reason: 'image_error' }
  // 不继续循环，直接退出
}
```

### 5. 用户中断（AbortController）

```typescript
if (toolUseContext.abortController.signal.aborted) {
  // 消费 StreamingToolExecutor 中剩余的工具
  // （生成合成的 tool_result 块，保持消息格式合法）
  for await (const update of streamingToolExecutor.getRemainingResults()) {
    if (update.message) yield update.message
  }
  
  // 非 submit-interrupt 才发送中断消息
  if (toolUseContext.abortController.signal.reason !== 'interrupt') {
    yield createUserInterruptionMessage({ toolUse: false })
  }
  return { reason: 'aborted_streaming' }
}
```

### Stop Hooks 集成

每次 AI 响应完成（无工具调用）后，系统会执行 stop hooks：

```typescript
const stopHookResult = yield* handleStopHooks(
  messagesForQuery, assistantMessages,
  systemPrompt, userContext, systemContext,
  toolUseContext, querySource, stopHookActive,
)

if (stopHookResult.preventContinuation) {
  return { reason: 'stop_hook_prevented' }
}

if (stopHookResult.blockingErrors.length > 0) {
  // Stop hook 注入了新的错误消息，继续循环处理
  state = {
    messages: [...messagesForQuery, ...assistantMessages, ...stopHookResult.blockingErrors],
    transition: { reason: 'stop_hook_blocking_error' },
    // ...
  }
  continue
}
```

**重要**：API 错误消息（rate limit、auth failure 等）会跳过 stop hooks，防止"错误 → hook 重试 → 错误"的死循环。

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
