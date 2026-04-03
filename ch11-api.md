# 第 11 章：Claude API 集成

## 11.1 API 客户端架构

[src/services/api/claude.ts](../../src/services/api/claude.ts)（125KB）是 Claude API 集成的核心，处理所有与 AI 模型的通信。

```
QueryEngine
    │
    ▼
callModel()  ← 主要 API 调用函数
    │
    ├─ 构建请求参数
    ├─ 选择 API 提供商（Anthropic/Bedrock/Vertex）
    ├─ 发起流式请求
    └─ 处理流式响应
```

---

## 11.2 多云平台支持

```typescript
// src/utils/model/providers.ts
export function getAPIProvider(model: string): APIProvider {
  if (model.startsWith('anthropic.')) return 'bedrock';
  if (model.startsWith('claude-') && isVertexModel(model)) return 'vertex';
  return 'anthropic';
}

// 根据提供商选择 SDK
const client = provider === 'bedrock'
  ? new AnthropicBedrock({ ... })
  : provider === 'vertex'
    ? new AnthropicVertex({ ... })
    : new Anthropic({ apiKey: getApiKey() });
```

---

## 11.3 流式响应处理

Claude API 使用 Server-Sent Events（SSE）流式返回响应：

```typescript
// 流式处理核心逻辑
for await (const event of stream) {
  switch (event.type) {
    case 'content_block_start':
      // 新内容块开始（文本或工具调用）
      if (event.content_block.type === 'tool_use') {
        // 工具调用开始，立即准备执行
        pendingToolCalls.set(event.index, {
          id: event.content_block.id,
          name: event.content_block.name,
          input: '',
        });
      }
      break;
      
    case 'content_block_delta':
      if (event.delta.type === 'text_delta') {
        // 文本增量，实时显示给用户
        yield { type: 'text_delta', text: event.delta.text };
      } else if (event.delta.type === 'input_json_delta') {
        // 工具调用参数增量，累积 JSON
        const tool = pendingToolCalls.get(event.index);
        if (tool) tool.input += event.delta.partial_json;
      }
      break;
      
    case 'content_block_stop':
      // 内容块结束，工具调用参数完整
      const tool = pendingToolCalls.get(event.index);
      if (tool) {
        yield { type: 'tool_use', ...tool, input: JSON.parse(tool.input) };
      }
      break;
      
    case 'message_stop':
      // 整个响应结束
      break;
  }
}
```

---

## 11.4 请求参数构建

```typescript
// 构建 API 请求参数
const requestParams: BetaMessageStreamParams = {
  model: resolvedModel,
  max_tokens: getModelMaxOutputTokens(model),
  
  // 系统提示（支持缓存）
  system: buildSystemPromptWithCaching(systemPrompt),
  
  // 消息历史
  messages: normalizeMessagesForAPI(messages),
  
  // 工具定义
  tools: tools.map(toolToAPISchema),
  
  // 思考模式配置
  ...(thinkingConfig.type !== 'disabled' ? {
    thinking: { type: 'enabled', budget_tokens: thinkingConfig.budgetTokens }
  } : {}),
  
  // Beta 功能
  betas: getMergedBetas(model),
};
```

---

## 11.5 Prompt 缓存

Claude API 支持 Prompt 缓存，减少重复内容的 Token 消耗。缓存控制逻辑在 [src/services/api/claude.ts](../../src/services/api/claude.ts) 中实现：

```typescript
// 缓存控制对象的构建
export function getCacheControl({
  scope,
  querySource,
}: {
  scope?: CacheScope
  querySource?: QuerySource
} = {}): { type: 'ephemeral'; ttl?: '1h'; scope?: 'global' } {
  return {
    type: 'ephemeral',
    // 1 小时 TTL：仅对特定用户类型和 querySource 白名单启用
    ...(should1hCacheTTL(querySource) && { ttl: '1h' }),
    // global scope：跨用户共享缓存（仅 Anthropic 内部）
    ...(scope === 'global' && { scope }),
  }
}
```

**1 小时 TTL 的启用条件**（`should1hCacheTTL`）：

```typescript
function should1hCacheTTL(querySource?: QuerySource): boolean {
  // Bedrock 用户需要显式环境变量
  if (getAPIProvider() === 'bedrock' && isEnvTruthy(process.env.ENABLE_PROMPT_CACHING_1H_BEDROCK))
    return true

  // 用户资格检查（claude.ai 订阅用户且未超额）
  let userEligible = getPromptCache1hEligible()
  if (userEligible === null) {
    userEligible = process.env.USER_TYPE === 'ant' ||
      (isClaudeAISubscriber() && !currentLimits.isUsingOverage)
    setPromptCache1hEligible(userEligible)
  }
  if (!userEligible) return false

  // querySource 白名单（通过 GrowthBook feature flag 控制）
  const config = getFeatureValue_CACHED_MAY_BE_STALE<{ allowlist?: string[] }>(
    'tengu_prompt_cache_1h_config', {}
  )
  const allowlist = config.allowlist ?? []
  return querySource !== undefined &&
    allowlist.some(pattern =>
      pattern.endsWith('*')
        ? querySource.startsWith(pattern.slice(0, -1))
        : querySource === pattern,
    )
}
```

**缓存禁用的环境变量**：

```
DISABLE_PROMPT_CACHING=1          → 全局禁用
DISABLE_PROMPT_CACHING_HAIKU=1    → 仅禁用 Haiku 模型的缓存
DISABLE_PROMPT_CACHING_SONNET=1   → 仅禁用 Sonnet 模型的缓存
DISABLE_PROMPT_CACHING_OPUS=1     → 仅禁用 Opus 模型的缓存
```

---

## 11.6 重试与错误处理

[src/services/api/withRetry.ts](../../src/services/api/withRetry.ts) 实现了完整的重试逻辑，并支持模型降级：

```typescript
// withRetry 的核心签名
export async function withRetry<T>(
  createClient: () => AnthropicClient,
  fn: (client: AnthropicClient) => Promise<T>,
  options: {
    maxRetries?: number
    model: string
    thinkingConfig: ThinkingConfig
  },
): Promise<T>
```

**FallbackTriggeredError**：当主模型不可用时，`withRetry` 抛出此错误，`queryLoop` 捕获后切换到备用模型：

```typescript
export class FallbackTriggeredError extends Error {
  constructor(
    public readonly originalModel: string,
    public readonly fallbackModel: string,
    public readonly originalError: unknown,
  ) { super('Model fallback triggered') }
}
```

**非流式回退**（`executeNonStreamingRequest`）：

当流式请求失败时，系统会尝试非流式请求作为回退：

```typescript
async function* executeNonStreamingRequest(
  anthropic: AnthropicClient,
  params: BetaMessageStreamParams,
): AsyncGenerator<BetaRawMessageStreamEvent> {
  // 120s 超时（远程模式），300s（默认）
  const timeout = isRemoteMode() ? 120_000 : 300_000

  try {
    const response = await anthropic.beta.messages.create({
      ...params,
      stream: false,
    }, { timeout })
    // 将非流式响应转换为流式事件格式
    yield* convertToStreamEvents(response)
  } catch (error) {
    logEvent('tengu_nonstreaming_fallback_error', { error: String(error) })
    throw error
  }
}
```

---

## 11.7 成本追踪

[src/cost-tracker.ts](../../src/cost-tracker.ts) 追踪每次 API 调用的 Token 使用量和费用：

```typescript
export function updateUsage(usage: NonNullableUsage, model: string): void {
  const cost = calculateCostFromTokens(
    usage.input_tokens,
    usage.output_tokens,
    usage.cache_creation_input_tokens,
    usage.cache_read_input_tokens,
    model,
  )
  addTotalCost(cost)
  addModelUsage(model, usage)
}
```

**Token 使用量的四个维度**：
- `input_tokens`：普通输入 token
- `output_tokens`：输出 token
- `cache_creation_input_tokens`：创建缓存的 token（首次缓存时计费）
- `cache_read_input_tokens`：读取缓存的 token（命中缓存时折扣计费）

---

## 11.8 Beta 功能与请求参数

系统通过 beta headers 启用实验性功能：

```typescript
// src/constants/betas.ts 中定义的 beta headers
const AFK_MODE_BETA_HEADER = 'afk-mode-2025-01-01'
const CONTEXT_1M_BETA_HEADER = 'max-tokens-3-5-sonnet-2024-07-15'
const EFFORT_BETA_HEADER = 'thinking-effort-2025-01-01'
const FAST_MODE_BETA_HEADER = 'fast-mode-2025-01-01'
const PROMPT_CACHING_BETA_HEADER = 'prompt-caching-2024-07-31'
const TASK_BUDGETS_BETA_HEADER = 'task-budgets-2025-01-01'
```

**Effort 参数配置**（控制思考深度）：

```typescript
function configureEffortParams(
  effortValue: EffortValue | undefined,
  outputConfig: BetaOutputConfig,
  extraBodyParams: Record<string, unknown>,
  betas: string[],
  model: string,
): void {
  if (!modelSupportsEffort(model) || 'effort' in outputConfig) return

  if (effortValue === undefined) {
    betas.push(EFFORT_BETA_HEADER)  // 启用 effort 但不指定值（使用模型默认）
  } else if (typeof effortValue === 'string') {
    outputConfig.effort = effortValue  // 'low' | 'medium' | 'high'
    betas.push(EFFORT_BETA_HEADER)
  } else if (process.env.USER_TYPE === 'ant') {
    // 内部用户可以传入数值 effort_override
    extraBodyParams.anthropic_internal = {
      ...extraBodyParams.anthropic_internal,
      effort_override: effortValue,
    }
  }
}
```

**Task Budget 参数**（控制 token 预算）：

```typescript
export function configureTaskBudgetParams(
  taskBudget: Options['taskBudget'],
  outputConfig: BetaOutputConfig & { task_budget?: TaskBudgetParam },
  betas: string[],
): void {
  if (!taskBudget || 'task_budget' in outputConfig) return

  outputConfig.task_budget = {
    type: 'tokens',
    total: taskBudget.total,
    ...(taskBudget.remaining !== undefined && { remaining: taskBudget.remaining }),
  }
  if (!betas.includes(TASK_BUDGETS_BETA_HEADER)) {
    betas.push(TASK_BUDGETS_BETA_HEADER)
  }
}
```

---

## 11.9 消息格式转换

内部消息类型与 API 消息类型之间的转换：

```typescript
// 内部 UserMessage → API MessageParam
function userMessageToMessageParam(
  message: UserMessage,
  enableCaching: boolean,
): MessageParam

// 内部 AssistantMessage → API MessageParam
function assistantMessageToMessageParam(
  message: AssistantMessage,
  enableCaching: boolean,
): MessageParam
```

**缓存标记的放置策略**：缓存控制标记放在消息历史的最后几个 `user` 消息上，确保最近的上下文被缓存，而不是整个历史。

---

## 小结

Claude API 集成层是整个系统的通信核心，它处理了流式响应、多云平台、Prompt 缓存、重试逻辑等复杂问题，为上层的 QueryEngine 提供了可靠的 AI 通信基础。
