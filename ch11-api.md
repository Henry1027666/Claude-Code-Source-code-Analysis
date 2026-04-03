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

Claude API 支持 Prompt 缓存，减少重复内容的 Token 消耗：

```typescript
// 系统提示缓存
function buildSystemPromptWithCaching(systemPrompt: SystemPrompt): TextBlockParam[] {
  return systemPrompt.map((part, index) => ({
    type: 'text',
    text: part,
    // 最后一个系统提示块标记为可缓存
    ...(index === systemPrompt.length - 1 ? {
      cache_control: { type: 'ephemeral' }
    } : {}),
  }));
}
```

---

## 11.6 重试与错误处理

[src/services/api/withRetry.ts](../../src/services/api/withRetry.ts)（28KB）实现了完整的重试逻辑：

```typescript
// 可重试的错误类型
const RETRYABLE_ERRORS = [
  'overloaded_error',    // 服务器过载
  'api_error',           // 临时 API 错误
  'timeout',             // 请求超时
];

// 指数退避重试
async function withRetry<T>(
  fn: () => Promise<T>,
  maxRetries = 3,
): Promise<T> {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (error) {
      if (attempt === maxRetries || !isRetryable(error)) throw error;
      
      // 指数退避：1s, 2s, 4s
      await sleep(Math.pow(2, attempt) * 1000);
    }
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
  );
  
  // 更新全局状态
  addTotalCost(cost);
  addModelUsage(model, usage);
}
```

---

## 11.8 Fast Mode

Fast Mode 使用相同的 Claude Opus 4.6 模型，但通过优化的请求参数提高响应速度：

```typescript
// src/utils/fastMode.ts
export function isFastModeEnabled(): boolean {
  return getGlobalConfig().fastMode === true;
}

// Fast Mode 下调整请求参数
if (isFastModeEnabled()) {
  requestParams.max_tokens = Math.min(requestParams.max_tokens, 4096);
  // 禁用思考模式以提高速度
  delete requestParams.thinking;
}
```

---

## 小结

Claude API 集成层是整个系统的通信核心，它处理了流式响应、多云平台、Prompt 缓存、重试逻辑等复杂问题，为上层的 QueryEngine 提供了可靠的 AI 通信基础。
