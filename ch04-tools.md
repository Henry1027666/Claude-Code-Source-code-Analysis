# 第 4 章：工具系统

## 4.1 工具框架设计

工具系统是 Claude Code 中最核心的扩展点。AI 与真实世界的所有交互——读文件、执行命令、搜索代码——都通过工具完成。

### Tool 接口定义

[src/Tool.ts:362](../../src/Tool.ts#L362) 定义了所有工具必须实现的接口：

```typescript
export type Tool<
  Input extends AnyObject = AnyObject,
  Output = unknown,
  P extends ToolProgressData = ToolProgressData,
> = {
  // 基本标识
  readonly name: string
  aliases?: string[]           // 向后兼容的别名
  searchHint?: string          // 工具搜索关键词提示
  
  // 核心方法
  call(
    args: z.infer<Input>,
    context: ToolUseContext,
    canUseTool: CanUseToolFn,
    parentMessage: AssistantMessage,
    onProgress?: ToolCallProgress<P>,
  ): Promise<ToolResult<Output>>
  
  // 权限相关
  checkPermissions(input, context): Promise<PermissionResult>
  validateInput?(input, context): Promise<ValidationResult>
  
  // 行为标志
  isConcurrencySafe(input): boolean   // 是否可以并行执行
  isReadOnly(input): boolean          // 是否只读
  isDestructive?(input): boolean      // 是否不可逆
  isEnabled(): boolean                // 是否启用
  
  // Schema
  readonly inputSchema: Input         // Zod 输入验证 Schema
  outputSchema?: z.ZodType<unknown>   // 输出 Schema
  
  // UI 相关
  description(input, options): Promise<string>
  userFacingName(input): string
  prompt(options): Promise<string>    // 工具的系统提示描述
  
  // 性能控制
  maxResultSizeChars: number          // 结果大小限制
  
  // 高级特性
  interruptBehavior?(): 'cancel' | 'block'  // 用户中断时的行为
  shouldDefer?: boolean               // 是否延迟加载
  alwaysLoad?: boolean                // 是否始终加载
  getToolUseSummary?(input): string | null  // 紧凑视图摘要
}
```

### ToolUseContext：工具执行上下文

每次工具调用都会收到一个 `ToolUseContext`，包含执行所需的所有上下文信息：

```typescript
export type ToolUseContext = {
  options: {
    tools: Tools              // 所有可用工具
    commands: Command[]       // 所有可用命令
    mcpClients: MCPServerConnection[]
    mainLoopModel: string
    thinkingConfig: ThinkingConfig
    // ...
  }
  abortController: AbortController  // 取消信号
  readFileState: FileStateCache      // 文件读取缓存
  getAppState(): AppState            // 读取 UI 状态
  setAppState(f): void               // 更新 UI 状态
  setToolJSX?: SetToolJSXFn          // 设置工具 UI
  agentId?: AgentId                  // 子 Agent ID（如果是子 Agent）
  messages: Message[]                // 当前对话历史
  // ...
}
```

---

## 4.2 工具分类与实现

### 文件操作工具（5 个）

#### FileEditTool：精确文件编辑

[src/tools/FileEditTool/FileEditTool.ts](../../src/tools/FileEditTool/FileEditTool.ts) 是最复杂的工具之一。它实现了精确的字符串替换：

```typescript
// 输入 Schema
const inputSchema = z.object({
  file_path: z.string(),    // 目标文件路径
  old_string: z.string(),   // 要替换的原始文本
  new_string: z.string(),   // 替换后的新文本
  replace_all: z.boolean().optional(),  // 是否替换所有匹配
})
```

**关键实现细节**：

1. **修改时间检查**：在执行编辑前检查文件修改时间，防止并发修改冲突
2. **相似文件建议**：如果文件不存在，会搜索相似文件名并提示
3. **行尾符保留**：检测并保留原文件的行尾符（CRLF/LF）
4. **Git Diff 生成**：编辑后生成 diff 用于 UI 展示
5. **LSP 通知**：通知语言服务器文件已更新
6. **技能发现**：检查编辑的文件路径是否触发新技能加载

```typescript
// 并发安全性：FileEditTool 不是并发安全的
isConcurrencySafe(input): boolean {
  return false;  // 写操作必须串行
}

// 不可逆操作标记
isDestructive(input): boolean {
  return false;  // 编辑可以撤销（通过 old_string 恢复）
}
```

#### FileReadTool：智能文件读取

支持多种文件类型：
- 普通文本文件（带行号）
- 图片（自动压缩，转 base64）
- PDF（支持页码范围）
- Jupyter Notebook（展示代码和输出）

```typescript
// 结果大小限制：Infinity（不持久化，工具自身控制大小）
maxResultSizeChars: Infinity
```

### 代码执行工具（3 个）

#### BashTool：Shell 命令执行

BashTool 是权限要求最严格的工具，因为它可以执行任意系统命令：

```typescript
// 并发安全性：Bash 命令不能并行（可能有副作用）
isConcurrencySafe(input): boolean {
  return false;
}

// 不可逆操作
isDestructive(input): boolean {
  // 检测危险命令：rm -rf, git push --force 等
  return containsDestructiveCommand(input.command);
}
```

**后台执行支持**：

```typescript
// 输入 Schema 包含 run_in_background 选项
const inputSchema = z.object({
  command: z.string(),
  description: z.string(),
  timeout: z.number().optional(),
  run_in_background: z.boolean().optional(),
})
```

当 `run_in_background: true` 时，命令在后台执行，不阻塞对话流程。

#### REPLTool：有状态交互式执行

REPLTool 维护一个持久的执行环境（如 Python 解释器），多次调用之间变量状态共享：

```typescript
// 透明包装器：REPL 的 UI 由内部工具调用的进度处理器渲染
isTransparentWrapper(): boolean {
  return true;
}
```

### Web 工具（2 个）

#### WebFetchTool：网页内容获取

```typescript
// 处理流程：
// 1. 发起 HTTP 请求
// 2. 将 HTML 转换为 Markdown（使用 Turndown）
// 3. 截断到合理大小
// 4. 返回给 AI

// 并发安全：只读操作，可以并行
isConcurrencySafe(input): boolean {
  return true;
}
```

### Agent 工具（1 个）

#### AgentTool：子 Agent 派发

[src/tools/AgentTool/](../../src/tools/AgentTool/) 是最复杂的工具目录，包含 15 个文件：

```typescript
// 内置 Agent 类型
const BUILT_IN_AGENTS = [
  'general-purpose',    // 通用 Agent
  'Explore',            // 代码库探索
  'Plan',               // 实现规划
  'claude-code-guide',  // Claude Code 使用指南
  'statusline-setup',   // 状态栏配置
]
```

**子 Agent 执行流程**：

```
AgentTool.call()
    │
    ▼
创建子 Agent 上下文（隔离的 ToolUseContext）
    │
    ▼
runAgent()
    │
    ├─ 构建子 Agent 系统提示
    ├─ 调用 query()（独立的查询循环）
    └─ 收集结果
    │
    ▼
返回子 Agent 的最终输出
```

**内存快照**：子 Agent 完成后，会保存内存快照用于后续恢复：

```typescript
// src/tools/AgentTool/agentMemorySnapshot.ts
await saveAgentMemorySnapshot(agentId, {
  messages: finalMessages,
  usage: totalUsage,
  // ...
});
```

---

## 4.3 工具注册与发现

[src/tools.ts](../../src/tools.ts) 负责工具的注册和管理：

```typescript
export function getTools(options: GetToolsOptions): Tools {
  const tools: Tool[] = [
    // 文件操作
    FileReadTool,
    FileWriteTool,
    FileEditTool,
    GlobTool,
    GrepTool,
    
    // 代码执行
    BashTool,
    ...(isWindows ? [PowerShellTool] : []),
    REPLTool,
    
    // Web
    WebFetchTool,
    WebSearchTool,
    
    // AI
    AgentTool,
    SkillTool,
    
    // 任务管理
    TaskCreateTool,
    TaskUpdateTool,
    TaskGetTool,
    TaskListTool,
    TaskStopTool,
    
    // MCP
    ...mcpTools,
    
    // 其他
    TodoWriteTool,
    AskUserQuestionTool,
    EnterPlanModeTool,
    ExitPlanModeTool,
    EnterWorktreeTool,
    ExitWorktreeTool,
    ScheduleCronTool,
    // ...
  ];
  
  return tools.filter(t => t.isEnabled());
}
```

### 工具延迟加载（Tool Deferral）

对于不常用的工具，系统支持延迟加载：

```typescript
// 工具标记为延迟加载
readonly shouldDefer = true;

// AI 需要先调用 ToolSearchTool 找到工具
// 然后才能使用该工具
```

这减少了系统提示的大小，提高了 AI 的响应质量。

---

## 4.4 工具结果大小控制

当工具返回大量内容时，系统会自动将结果持久化到磁盘：

```typescript
// src/utils/toolResultStorage.ts
export async function applyToolResultBudget(
  messages: Message[],
  contentReplacementState: ContentReplacementState | undefined,
  persistRecords: ((records) => void) | undefined,
  noBudgetTools: Set<string>,
): Promise<Message[]> {
  // 对超过 maxResultSizeChars 的工具结果：
  // 1. 将完整内容写入临时文件
  // 2. 在消息中替换为文件路径引用
  // 3. AI 可以通过 FileReadTool 读取完整内容
}
```

---

## 4.5 工具权限检查流程

每次工具调用前，都会经过权限检查：

```
工具调用请求
    │
    ▼
tool.validateInput()     ← 输入格式验证（Zod）
    │
    ▼
tool.checkPermissions()  ← 工具特定权限检查
    │
    ▼
canUseTool()             ← 全局权限系统检查
    │
    ├─ 自动允许 → 直接执行
    ├─ 询问用户 → 显示权限对话框
    └─ 拒绝 → 返回错误给 AI
```

---

## 4.6 MCP 工具集成

MCP（Model Context Protocol）工具是动态注册的外部工具：

```typescript
// src/tools/MCPTool/MCPTool.ts
export function createMCPTool(
  serverName: string,
  toolName: string,
  description: string,
  inputSchema: ToolInputJSONSchema,
  mcpClient: MCPServerConnection,
): Tool {
  return {
    name: `mcp__${serverName}__${toolName}`,
    isMcp: true,
    
    async call(args, context) {
      // 通过 MCP 协议调用外部工具
      const result = await mcpClient.callTool(toolName, args);
      return { data: result };
    },
    
    // MCP 工具使用 JSON Schema 而非 Zod
    inputJSONSchema: inputSchema,
    
    // 并发安全性由工具元数据决定
    isConcurrencySafe: () => !toolMeta.isDestructive,
  };
}
```

---

## 4.7 工具 UI 渲染

每个工具都有对应的 UI 渲染逻辑，用于在终端中展示工具调用状态：

```typescript
// 工具可以设置自定义 JSX 渲染
context.setToolJSX?.({
  jsx: <FileEditDisplay diff={diff} />,
  shouldHidePromptInput: false,
  showSpinner: true,
});

// 工具完成后清除 JSX
context.setToolJSX?.(null);
```

工具的 UI 状态通过 `setInProgressToolUseIDs` 追踪：

```typescript
// 工具开始执行
context.setInProgressToolUseIDs(prev => new Set([...prev, toolUseId]));

// 工具执行完成
context.setInProgressToolUseIDs(prev => {
  const next = new Set(prev);
  next.delete(toolUseId);
  return next;
});
```

---

## 4.8 工具设计模式总结

Claude Code 的工具系统体现了几个关键设计模式：

### 1. 接口驱动设计
所有工具实现同一个 `Tool` 接口，使得工具可以统一处理（权限检查、并发控制、结果限制）。

### 2. Zod Schema 验证
每个工具的输入都有严格的 Zod Schema，在执行前验证参数格式，防止 AI 传入错误参数。

### 3. 并发安全标记
工具通过 `isConcurrencySafe()` 声明自己是否可以并行执行，`StreamingToolExecutor` 据此决定执行策略。

### 4. 结果大小控制
`maxResultSizeChars` 防止工具返回过大的结果撑爆上下文窗口，超限时自动持久化到磁盘。

### 5. 渐进式权限
工具通过 `checkPermissions()` 实现细粒度的权限控制，而不是简单的"允许/拒绝"。

---

## 小结

工具系统是 Claude Code 的核心扩展机制。45 个内置工具覆盖了软件开发的主要场景，而 MCP 协议允许无限扩展。工具框架的设计确保了：

- **安全性**：每次调用都经过权限检查
- **可靠性**：Zod Schema 验证防止参数错误
- **性能**：并发安全标记实现最优执行策略
- **可扩展性**：统一接口使新工具易于添加

下一章，我们将深入权限系统——这是保护用户系统安全的核心机制。
