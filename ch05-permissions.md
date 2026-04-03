# 第 5 章：权限系统

## 5.1 设计哲学

Claude Code 能够执行真实的系统命令——删除文件、运行脚本、推送代码。这种能力需要一套严格的安全护栏。权限系统的核心原则是：

**默认拒绝，明确允许（Default Deny, Explicit Allow）**

每次工具调用都必须通过权限检查，未经明确授权的操作会被拦截并询问用户。

---

## 5.2 权限模式

[src/utils/permissions/PermissionMode.ts](../../src/utils/permissions/PermissionMode.ts) 定义了六种权限模式：

```typescript
const PERMISSION_MODE_CONFIG = {
  default: {
    title: 'Default',
    symbol: '',
    color: 'text',
    // 标准模式：遇到未知操作询问用户
  },
  plan: {
    title: 'Plan Mode',
    symbol: '⏸',
    color: 'planMode',
    // 计划模式：只允许只读操作，写操作需要用户确认
  },
  acceptEdits: {
    title: 'Accept edits',
    symbol: '⏵⏵',
    color: 'autoAccept',
    // 自动接受工作目录内的文件编辑
  },
  bypassPermissions: {
    title: 'Bypass Permissions',
    symbol: '⏵⏵',
    color: 'error',  // 红色警告！
    // 跳过所有权限检查（危险！）
  },
  dontAsk: {
    title: "Don't Ask",
    symbol: '⏵⏵',
    color: 'error',
    // 遇到需要询问的操作直接拒绝
  },
  auto: {
    title: 'Auto mode',
    symbol: '⏵⏵',
    color: 'warning',
    // Anthropic 内部：用 AI 分类器自动判断
  },
}
```

### 模式选择建议

| 场景 | 推荐模式 |
|------|---------|
| 日常开发 | `default` |
| 代码审查（只读） | `dontAsk` |
| 快速迭代（信任 AI） | `acceptEdits` |
| CI/CD 自动化 | `bypassPermissions` |
| 复杂任务规划 | `plan` |

---

## 5.3 规则系统

### 规则类型

每条权限规则属于三种类型之一：

```typescript
type PermissionBehavior = 'allow' | 'deny' | 'ask'

// 规则格式示例：
// "Bash"              → 匹配整个 Bash 工具
// "Bash(git *)"       → 匹配 Bash 工具中以 git 开头的命令
// "mcp__server1"      → 匹配 MCP 服务器 server1 的所有工具
// "mcp__server1__*"   → 同上（通配符形式）
```

### 规则存储层级

规则按优先级从高到低存储在不同位置：

```
优先级（高 → 低）:
┌─────────────────────────────────────────────────────┐
│ 1. .claude/policy.json      ← 管理员策略（只读）     │
│ 2. ~/.claude/settings.json  ← 用户全局设置           │
│ 3. .claude/settings.json    ← 项目设置               │
│ 4. .claude/local.json       ← 本地覆盖（不提交 git） │
│ 5. 命令行参数               ← --permission-allow Bash │
│ 6. 当前会话                 ← 仅本次有效             │
└─────────────────────────────────────────────────────┘
```

**示例 settings.json：**

```json
{
  "permissions": {
    "alwaysAllow": [
      "Bash(git status)",
      "Bash(git diff*)",
      "Read",
      "Glob",
      "Grep"
    ],
    "alwaysDeny": [
      "Bash(rm -rf*)",
      "Bash(git push --force*)"
    ],
    "alwaysAsk": [
      "Bash(npm publish*)",
      "Bash(pip install*)"
    ]
  }
}
```

---

## 5.4 权限检查流程

[src/utils/permissions/permissions.ts](../../src/utils/permissions/permissions.ts) 实现了完整的权限检查逻辑：

```
工具调用请求
    │
    ▼
[1] 命中 Deny 规则？
    ├─ 是 → 直接拒绝（即使在 bypassPermissions 模式下也无法绕过）
    └─ 否 ↓
    
[2] 命中 Ask 规则？
    ├─ 是 → 强制询问用户（即使在 bypassPermissions 模式下也无法绕过）
    └─ 否 ↓
    
[3] 工具自检（tool.checkPermissions()）
    ├─ 拒绝 → 返回拒绝原因
    └─ 通过 ↓
    
[4] 当前是 bypassPermissions 模式？
    ├─ 是 → 直接允许
    └─ 否 ↓
    
[5] 命中 Allow 规则？
    ├─ 是 → 直接允许
    └─ 否 ↓
    
[6] 默认行为：询问用户
    │
    ▼
[7] auto 模式？
    ├─ 是 → 先运行 AI 分类器
    │       ├─ 分类器：安全 → 允许
    │       ├─ 分类器：危险 → 记录拒绝次数
    │       └─ 拒绝次数超限 → 降级为询问用户
    └─ 否 → 显示权限对话框
```

### 关键代码实现

```typescript
// src/utils/permissions/permissions.ts（简化版）
export async function canUseTool(
  tool: Tool,
  input: unknown,
  context: ToolUseContext,
  assistantMessage: AssistantMessage,
  toolUseID: string,
  forceDecision?: 'allow' | 'deny',
): Promise<PermissionResult> {
  const permContext = context.getAppState().toolPermissionContext;
  
  // 1. 检查 Deny 规则（最高优先级，不可绕过）
  const denyRule = getDenyRuleForTool(permContext, tool);
  if (denyRule) {
    return { behavior: 'deny', reason: { type: 'rule', rule: denyRule } };
  }
  
  // 2. 检查 Ask 规则（强制询问，不可绕过）
  const askRule = getAskRuleForTool(permContext, tool);
  if (askRule) {
    return { behavior: 'ask', reason: { type: 'rule', rule: askRule } };
  }
  
  // 3. 工具自检
  const toolPermResult = await tool.checkPermissions(input, context);
  if (toolPermResult.behavior === 'deny') return toolPermResult;
  
  // 4. bypassPermissions 模式
  if (permContext.mode === 'bypassPermissions') {
    return { behavior: 'allow' };
  }
  
  // 5. Allow 规则
  const allowRule = toolAlwaysAllowedRule(permContext, tool);
  if (allowRule) {
    return { behavior: 'allow', reason: { type: 'rule', rule: allowRule } };
  }
  
  // 6. 默认：询问用户
  return { behavior: 'ask' };
}
```

---

## 5.5 规则匹配算法

[src/utils/permissions/shellRuleMatching.ts](../../src/utils/permissions/shellRuleMatching.ts) 实现了规则匹配逻辑：

```typescript
// 规则格式：ToolName(pattern)
// 例如：Bash(git *)

export function matchWildcardPattern(pattern: string, value: string): boolean {
  // 支持 * 通配符
  // "git *" 匹配 "git status", "git diff HEAD" 等
  // "npm publish*" 匹配 "npm publish", "npm publish --dry-run" 等
}
```

### MCP 工具规则匹配

MCP 工具有特殊的命名规则：

```typescript
// MCP 工具名格式：mcp__serverName__toolName
// 规则可以匹配：
// - "mcp__server1"        → 匹配 server1 的所有工具
// - "mcp__server1__*"     → 同上（通配符形式）
// - "mcp__server1__write" → 只匹配 server1 的 write 工具

function toolMatchesRule(tool, rule): boolean {
  const ruleInfo = mcpInfoFromString(rule.ruleValue.toolName);
  const toolInfo = mcpInfoFromString(tool.name);
  
  return (
    ruleInfo !== null &&
    toolInfo !== null &&
    (ruleInfo.toolName === undefined || ruleInfo.toolName === '*') &&
    ruleInfo.serverName === toolInfo.serverName
  );
}
```

---

## 5.6 拒绝次数追踪

[src/utils/permissions/denialTracking.ts](../../src/utils/permissions/denialTracking.ts) 防止 AI 分类器无限自动拒绝：

```typescript
export const DENIAL_LIMITS = {
  maxConsecutive: 3,   // 连续拒绝 3 次 → 降级为询问用户
  maxTotal: 20,        // 总共拒绝 20 次 → 降级为询问用户
} as const;

export type DenialTrackingState = {
  consecutiveDenials: number;
  totalDenials: number;
};

export function shouldFallbackToPrompting(state: DenialTrackingState): boolean {
  return (
    state.consecutiveDenials >= DENIAL_LIMITS.maxConsecutive ||
    state.totalDenials >= DENIAL_LIMITS.maxTotal
  );
}
```

**设计意图**：防止 AI 分类器过于保守，导致用户体验变差。当分类器连续拒绝 3 次或总共拒绝 20 次后，系统会降级为询问用户，让用户自己决定。

---

## 5.7 Bash 命令分类器

对于 Bash 工具，系统有专门的 AI 分类器在后台判断命令是否危险：

```typescript
// src/utils/permissions/bashClassifier.ts（概念示意）
async function classifyBashCommand(command: string): Promise<'safe' | 'dangerous'> {
  // 检查危险模式：
  // - rm -rf / 或 rm -rf ~
  // - dd if=/dev/zero
  // - git push --force
  // - npm publish（发布包）
  // - curl | bash（管道执行）
  // - 命令注入（; rm -rf）
}
```

分类器与用户对话框**竞速**：

```typescript
const result = await Promise.race([
  classifyBashCommand(command),
  showPermissionDialog(tool, input),
]);

// 分类器先出结果：自动处理
// 用户先回答：按用户决定
```

---

## 5.8 权限持久化

用户在对话框中选择"永远允许"时，规则会被持久化：

```typescript
// src/utils/permissions/PermissionUpdate.ts
export async function persistPermissionUpdates(
  updates: PermissionUpdate[],
  destination: PermissionUpdateDestination,
): Promise<void> {
  // destination 可以是：
  // - 'localSettings'  → .claude/local.json（不提交 git）
  // - 'projectSettings' → .claude/settings.json（提交 git）
  // - 'globalSettings'  → ~/.claude/settings.json（全局）
  
  for (const update of updates) {
    await applyPermissionUpdate(update, destination);
  }
}
```

---

## 5.9 企业策略（MDM）

企业用户可以通过 MDM（Mobile Device Management）强制执行权限策略：

```
.claude/policy.json（管理员控制，只读）:
{
  "permissions": {
    "alwaysDeny": [
      "Bash(curl*)",
      "Bash(wget*)",
      "WebFetch"
    ]
  }
}
```

这些规则具有最高优先级，用户无法覆盖。

---

## 5.10 权限对话框 UI

当需要询问用户时，系统显示权限对话框：

```
┌─────────────────────────────────────────────────────┐
│  Claude wants to run a Bash command:                │
│                                                     │
│  $ git push origin main                             │
│                                                     │
│  [Allow once]  [Always allow]  [Deny]               │
└─────────────────────────────────────────────────────┘
```

对话框选项：
- **Allow once**：本次允许，下次还问
- **Always allow**：保存规则，以后不再问（写入 settings.json）
- **Deny**：本次拒绝，AI 收到拒绝消息

---

## 5.11 沙箱支持

对于高风险操作，系统支持沙箱执行：

```typescript
// src/tools/BashTool/shouldUseSandbox.ts
export function shouldUseSandbox(
  command: string,
  context: ToolUseContext,
): boolean {
  // 检查是否启用了沙箱
  // 检查命令是否需要沙箱隔离
}
```

沙箱通过 `SandboxManager` 管理，提供额外的系统隔离层。

---

## 小结

权限系统是 Claude Code 安全性的核心保障，其设计体现了几个关键原则：

1. **不可绕过的安全底线**：Deny 和 Ask 规则即使在 `bypassPermissions` 模式下也生效
2. **分层规则系统**：从管理员策略到会话级规则，灵活且可控
3. **智能分类器**：减少用户打扰，同时保持安全性
4. **防止过度拒绝**：拒绝次数追踪确保用户体验不会因分类器过于保守而变差
5. **透明的权限请求**：对话框清晰说明工具要做什么，用户可以做出知情决策

下一章，我们将深入认证系统——Claude Code 如何安全地管理 API 密钥和 OAuth 凭证。
