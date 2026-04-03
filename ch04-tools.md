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

[src/tools/FileEditTool/FileEditTool.ts](../../src/tools/FileEditTool/FileEditTool.ts) 是最复杂的工具之一，实现了精确的字符串替换。

**输入 Schema（严格模式）**：

```typescript
const inputSchema = z.strictObject({
  file_path: z.string(),
  old_string: z.string(),
  new_string: z.string(),
  replace_all: z.boolean().optional().default(false),
})
```

**`validateInput` 的安全检查链**（按顺序执行，任一失败即返回）：

```typescript
async validateInput(input, toolUseContext) {
  const { file_path, old_string, new_string, replace_all } = input
  const fullFilePath = expandPath(file_path)  // 展开 ~ 和相对路径

  // 1. 团队内存秘密检查（防止写入密钥）
  const secretError = checkTeamMemSecrets(fullFilePath, new_string)
  if (secretError) return { result: false, message: secretError, errorCode: 0 }

  // 2. 无变更检查
  if (old_string === new_string)
    return { result: false, behavior: 'ask', message: 'No changes to make...', errorCode: 1 }

  // 3. Deny 规则检查
  const denyRule = matchingRuleForInput(fullFilePath, appState.toolPermissionContext, 'edit', 'deny')
  if (denyRule !== null) return { result: false, behavior: 'ask', message: '...', errorCode: 2 }

  // 4. UNC 路径安全检查（Windows NTLM 凭证泄露防护）
  if (fullFilePath.startsWith('\\\\') || fullFilePath.startsWith('//'))
    return { result: true }  // 跳过文件系统操作，交给权限检查处理

  // 5. 文件大小检查（防止 OOM）
  const { size } = await fs.stat(fullFilePath)
  if (size > MAX_EDIT_FILE_SIZE)  // MAX_EDIT_FILE_SIZE = 1 GiB
    return { result: false, behavior: 'ask', message: `File too large...`, errorCode: 10 }
}
```

**`call` 的执行流程**：

```
1. 读取文件内容（readFileSyncWithMetadata，检测编码和行尾符）
2. 检查文件修改时间（防止并发修改冲突）
3. 检查是否为 Jupyter Notebook（.ipynb → 拒绝，提示用 NotebookEditTool）
4. 执行字符串替换（findActualString + 引号风格保留）
5. 写入磁盘（writeTextContent，保留原始行尾符）
6. 生成 Git Diff（fetchSingleFileGitDiff，用于 UI 展示）
7. 通知 LSP 服务器（notifyVscodeFileUpdated）
8. 记录文件历史（fileHistoryTrackEdit）
9. 触发技能发现（activateConditionalSkillsForPaths）
10. 清除 LSP 诊断缓存（clearDeliveredDiagnosticsForFile）
```

**字符串匹配的容错机制**（`findActualString`）：

当 `old_string` 在文件中找不到精确匹配时，工具会尝试：
1. 规范化空白字符后重新匹配
2. 保留原始引号风格（`preserveQuoteStyle`）
3. 如果仍找不到，返回详细错误（包含文件中相似内容的提示）

#### FileReadTool：智能文件读取

支持多种文件类型，通过输出类型区分：

```typescript
// 输出 Schema 是一个 union，根据文件类型返回不同结构
z.union([
  z.object({ type: z.literal('text'),     file: { content, lineCount, ... } }),
  z.object({ type: z.literal('image'),    file: { base64, mediaType, ... } }),
  z.object({ type: z.literal('notebook'), file: { filePath, cells } }),
  z.object({ type: z.literal('pdf'),      file: { filePath, base64, originalSize } }),
  z.object({ type: z.literal('parts'),    file: { filePath, count, outputDir } }),
  z.object({ type: z.literal('file_unchanged'), file: { filePath } }),
])
```

**`validateInput` 的安全检查**：

```typescript
// 阻止读取会挂起的设备文件
const BLOCKED_DEVICE_PATHS = [
  '/dev/zero', '/dev/random', '/dev/urandom',
  '/dev/stdin', '/dev/tty', '/dev/console',
  // ...
]
if (isBlockedDevicePath(fullFilePath))
  return { result: false, message: `Cannot read '${file_path}': this device file would block...`, errorCode: 9 }

// 阻止读取二进制文件（PDF、图片除外）
if (hasBinaryExtension(fullFilePath) && !isPDFExtension(ext) && !IMAGE_EXTENSIONS.has(ext))
  return { result: false, message: `This tool cannot read binary files...`, errorCode: 4 }
```

**关键设计**：`maxResultSizeChars: Infinity`——FileReadTool 不受全局结果大小限制，由工具自身通过 `maxTokens` 参数控制输出大小。

### 代码执行工具（3 个）

#### BashTool：Shell 命令执行

**完整输入 Schema**：

```typescript
const inputSchema = z.strictObject({
  command: z.string(),
  timeout: z.number().optional(),           // 最大超时（毫秒）
  description: z.string().optional(),       // 命令的语义描述（用于 UI 折叠）
  run_in_background: z.boolean().optional(),
  dangerouslyDisableSandbox: z.boolean().optional(),
  // 内部字段：sed 编辑预览结果（跳过实际执行）
  _simulatedSedEdit: z.object({ filePath, newContent }).optional(),
})
```

**命令分类系统**（用于 UI 折叠显示）：

```typescript
// 这些集合决定命令在 UI 中如何折叠
const BASH_SEARCH_COMMANDS = new Set(['find', 'grep', 'rg', 'ag', 'ack', 'locate', 'which', 'whereis'])
const BASH_READ_COMMANDS   = new Set(['cat', 'head', 'tail', 'less', 'wc', 'stat', 'jq', 'awk', 'cut', 'sort'])
const BASH_LIST_COMMANDS   = new Set(['ls', 'tree', 'du'])
const BASH_SILENT_COMMANDS = new Set(['mv', 'cp', 'rm', 'mkdir', 'chmod', 'touch', 'ln'])
```

**`call` 的执行流程**：

```
1. 处理 _simulatedSedEdit（直接应用，不执行 shell）
2. 调用 runShellCommand()（async generator，流式输出）
   ├─ 检查沙箱模式（shouldUseSandbox）
   ├─ 设置超时（默认或自定义）
   └─ 流式 yield 进度（onProgress 回调）
3. 消费 generator，收集 stdout/stderr
4. 处理自动后台化（assistant 模式下超过 15s 自动后台）
5. 语义化解释退出码（returnCodeInterpretation）
6. 持久化大输出（> 64MB 截断，写入 tool-results 目录）
7. 提取 Claude Code hints（从输出中解析特殊标记）
8. 处理图像输出（resize 后返回 base64）
9. 记录遥测（命令类型、退出码、是否使用代码索引工具）
```

**完整输出 Schema**：

```typescript
z.object({
  stdout: z.string(),
  stderr: z.string(),
  interrupted: z.boolean(),
  rawOutputPath: z.string().optional(),          // 大输出的磁盘路径
  isImage: z.boolean().optional(),
  backgroundTaskId: z.string().optional(),
  assistantAutoBackgrounded: z.boolean().optional(),
  dangerouslyDisableSandbox: z.boolean().optional(),
  returnCodeInterpretation: z.string().optional(),
  noOutputExpected: z.boolean().optional(),
  persistedOutputPath: z.string().optional(),
  persistedOutputSize: z.number().optional(),
})
```

#### REPLTool：有状态交互式执行

REPLTool 维护一个持久的执行环境（如 Python 解释器），多次调用之间变量状态共享。它是一个**透明包装器**——UI 渲染由内部工具调用的进度处理器负责，REPLTool 本身不渲染。

### Web 工具（2 个）

#### WebFetchTool：网页内容获取

处理流程：
1. 发起 HTTP 请求（带重定向跟踪）
2. 将 HTML 转换为 Markdown（使用 Turndown）
3. 截断到合理大小后返回给 AI

`isConcurrencySafe()` 返回 `true`——纯只读操作，可与其他只读工具并行执行。

### Agent 工具（1 个）

#### AgentTool：子 Agent 派发

[src/tools/AgentTool/](../../src/tools/AgentTool/) 是最复杂的工具目录。

**完整输入 Schema**：

```typescript
z.object({
  description: z.string(),          // 3-5 词的任务描述
  prompt: z.string(),               // 完整任务提示
  subagent_type: z.string().optional(),
  model: z.enum(['sonnet', 'opus', 'haiku']).optional(),
  run_in_background: z.boolean().optional(),
  // 多 Agent 扩展字段
  name: z.string().optional(),
  team_name: z.string().optional(),
  mode: permissionModeSchema().optional(),
  isolation: z.enum(['worktree']).optional(),  // 外部构建为 ['worktree', 'remote']
  cwd: z.string().optional(),
})
```

**输出 Schema（union）**：

```typescript
z.union([
  // 同步完成
  z.object({ status: z.literal('completed'), prompt, ...agentToolResultSchema() }),
  // 异步后台启动
  z.object({
    status: z.literal('async_launched'),
    agentId: z.string(),
    description: z.string(),
    prompt: z.string(),
    outputFile: z.string(),          // 检查进度的输出文件路径
    canReadOutputFile: z.boolean().optional(),
  }),
])
```

**`prompt()` 的动态生成**：

```typescript
async prompt({ agents, tools, getToolPermissionContext, allowedAgentTypes }) {
  // 1. 收集已配置 MCP 服务器的工具
  const mcpServersWithTools = tools
    .filter(t => t.name?.startsWith('mcp__'))
    .map(t => t.name.split('__')[1])

  // 2. 过滤掉 MCP 需求未满足的 Agent
  const agentsWithMcpRequirementsMet = filterAgentsByMcpRequirements(agents, mcpServersWithTools)

  // 3. 过滤掉被权限规则拒绝的 Agent
  const filteredAgents = filterDeniedAgents(agentsWithMcpRequirementsMet, toolPermissionContext, AGENT_TOOL_NAME)

  // 4. 生成系统提示（包含可用 Agent 列表）
  return await getPrompt(filteredAgents, isCoordinator, allowedAgentTypes)
}
```

**子 Agent 执行流程**：

```
AgentTool.call()
    │
    ├─ 检查 isolation: 'worktree' → 创建 git worktree 隔离环境
    ├─ 检查 run_in_background → 超过 120s 自动后台化
    │
    ▼
LocalAgentTask / RemoteAgentTask
    │
    ├─ 创建隔离的 ToolUseContext
    │   ├─ setAppState 替换为 no-op（防止修改主线程 UI 状态）
    │   ├─ 独立的 readFileState 缓存
    │   └─ 独立的 contentReplacementState
    │
    ├─ 调用 query()（独立的 queryLoop）
    │
    └─ 保存内存快照（agentMemorySnapshot）
           ├─ messages: finalMessages
           └─ usage: totalUsage
```

---

## 4.3 工具注册与发现

[src/tools.ts](../../src/tools.ts) 负责工具的注册和管理：

```typescript
export function getTools(options: GetToolsOptions): Tools {
  const tools: Tool[] = [
    // 文件操作
    FileReadTool, FileWriteTool, FileEditTool, GlobTool, GrepTool,
    // 代码执行
    BashTool,
    ...(isWindows ? [PowerShellTool] : []),
    REPLTool,
    // Web
    WebFetchTool, WebSearchTool,
    // AI
    AgentTool, SkillTool,
    // 任务管理
    TaskCreateTool, TaskUpdateTool, TaskGetTool, TaskListTool, TaskStopTool, TaskOutputTool,
    // MCP
    ...mcpTools,
    // 工作流
    TodoWriteTool, AskUserQuestionTool,
    EnterPlanModeTool, ExitPlanModeTool,
    EnterWorktreeTool, ExitWorktreeTool,
    ScheduleCronTool, SendMessageTool,
    // ...
  ]
  return tools.filter(t => t.isEnabled())
}
```

### 工具延迟加载（Tool Deferral）

对于不常用的工具，系统支持延迟加载——工具标记 `shouldDefer = true` 后不会出现在初始系统提示中，AI 需要先调用 `ToolSearchTool` 找到工具，再使用它。这减少了系统提示的大小，提高响应质量。

### buildTool 工厂函数

所有工具通过 `buildTool()` 工厂函数创建，它负责：
- 将 `ToolDef` 对象转换为完整的 `Tool` 实现
- 注入默认的 `isConcurrencySafe`、`isReadOnly`、`isEnabled` 实现
- 包装 `call()` 以添加统一的错误处理和遥测

```typescript
// src/Tool.ts
export function buildTool<Input, Output, Progress>(def: ToolDef<Input, Output, Progress>): Tool<Input, Output, Progress>
```

---

## 4.4 工具结果大小控制

当工具返回大量内容时，系统会自动将结果持久化到磁盘：

```typescript
// src/utils/toolResultStorage.ts
export async function applyToolResultBudget(
  messages: Message[],
  contentReplacementState: ContentReplacementState | undefined,
  persistRecords: ((records) => void) | undefined,
  noBudgetTools: Set<string>,  // 豁免工具（如 FileReadTool）
): Promise<Message[]>
```

**工作原理**：
1. 扫描消息历史中所有工具结果的内容大小
2. 对超过 `maxResultSizeChars` 的结果：将完整内容写入临时文件
3. 在消息中用文件路径引用替换原始内容
4. AI 可以通过 `FileReadTool` 读取完整内容

**持久化策略**：
- `agent:*` 来源（子 Agent）→ 写入 sidechain 文件（支持 AgentTool resume）
- `repl_main_thread` 来源 → 写入 session 文件（支持 `/resume`）
- 其他临时来源（如 agent_summary）→ 不持久化

## 4.5 工具权限检查流程

每次工具调用前，都会经过权限检查：

```
工具调用请求
    │
    ▼
tool.validateInput()     ← 输入格式验证（Zod Schema + 自定义逻辑）
    │
    ▼
tool.checkPermissions()  ← 工具特定权限检查（如文件路径白名单）
    │
    ▼
canUseTool()             ← 全局权限系统检查（见第 5 章）
    │
    ├─ allow → 直接执行
    ├─ ask   → 显示权限对话框，等待用户决定
    └─ deny  → 返回错误给 AI（AI 可以调整策略重试）
```

**`checkPermissions` 的工具特定逻辑**：

```typescript
// FileReadTool 的权限检查
async checkPermissions(input, context): Promise<PermissionDecision> {
  const appState = context.getAppState()
  return checkReadPermissionForTool(FileReadTool, input, appState.toolPermissionContext)
}

// FileEditTool 的权限检查
async checkPermissions(input, context): Promise<PermissionDecision> {
  const appState = context.getAppState()
  return checkWritePermissionForTool(FileEditTool, input, appState.toolPermissionContext)
}
```

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
      const result = await mcpClient.callTool(toolName, args)
      return { data: result }
    },
    // MCP 工具使用 JSON Schema 而非 Zod
    inputJSONSchema: inputSchema,
    // 并发安全性由工具元数据决定
    isConcurrencySafe: () => !toolMeta.isDestructive,
  }
}
```

## 4.7 工具 UI 渲染

每个工具都有对应的 UI 渲染逻辑，用于在终端中展示工具调用状态：

```typescript
// 工具可以设置自定义 JSX 渲染
context.setToolJSX?.({
  jsx: <FileEditDisplay diff={diff} />,
  shouldHidePromptInput: false,
  showSpinner: true,
})

// 工具完成后清除 JSX
context.setToolJSX?.(null)
```

工具的 UI 状态通过 `setInProgressToolUseIDs` 追踪：

```typescript
// 工具开始执行
context.setInProgressToolUseIDs(prev => new Set([...prev, toolUseId]))

// 工具执行完成
context.setInProgressToolUseIDs(prev => {
  const next = new Set(prev)
  next.delete(toolUseId)
  return next
})
```

每个工具还实现了以下 UI 相关方法：
- `renderToolUseMessage(input)` — 工具调用时显示的消息（如 `Edit src/foo.ts`）
- `renderToolResultMessage(output)` — 工具结果显示（如 diff 视图）
- `renderToolUseErrorMessage(error)` — 错误显示
- `getToolUseSummary(input)` — 紧凑视图中的摘要文本
- `getActivityDescription(input)` — 进度条中的活动描述（如 `Editing src/foo.ts`）

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
