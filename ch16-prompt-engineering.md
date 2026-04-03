# 第 16 章：提示词工程艺术

Claude Code 的提示词系统是整个项目中最精心设计的部分之一。本章深入分析 [src/constants/prompts.ts](../../src/constants/prompts.ts)（915 行）及相关文件，总结其提示词的规范、风格与工程技巧。

---

## 16.1 系统提示的整体架构

系统提示不是一个静态字符串，而是由多个**独立计算的片段**动态拼接而成：

```
getSystemPrompt()
    │
    ├─ [静态区域，全局可缓存]
    │   ├─ getSimpleIntroSection()       ← 身份与角色定义
    │   ├─ getSimpleSystemSection()      ← 工具执行规则
    │   ├─ getSimpleDoingTasksSection()  ← 任务执行规范
    │   ├─ getActionsSection()           ← 危险操作守则
    │   ├─ getUsingYourToolsSection()    ← 工具使用偏好
    │   ├─ getSimpleToneAndStyleSection() ← 语气与格式
    │   └─ getOutputEfficiencySection()  ← 输出效率要求
    │
    ├─ [SYSTEM_PROMPT_DYNAMIC_BOUNDARY] ← 缓存边界标记
    │
    └─ [动态区域，按会话计算]
        ├─ session_guidance              ← 会话特定指导
        ├─ memory                        ← 内存内容注入
        ├─ env_info_simple               ← 环境信息
        ├─ language                      ← 语言偏好
        ├─ output_style                  ← 输出风格配置
        ├─ mcp_instructions              ← MCP 服务器指令
        └─ scratchpad / frc / token_budget 等
```

### 缓存边界的工程意义

```typescript
// src/constants/prompts.ts
export const SYSTEM_PROMPT_DYNAMIC_BOUNDARY = '__SYSTEM_PROMPT_DYNAMIC_BOUNDARY__'

// 边界之前：scope: 'global'，跨用户共享缓存
// 边界之后：用户/会话特定，不可跨用户缓存
```

这个设计将系统提示分为两个缓存域：静态区域（所有用户共享同一缓存）和动态区域（每个用户独立缓存）。这大幅降低了 Token 消耗，因为静态区域只需缓存一次。

---

## 16.2 片段缓存系统

[src/constants/systemPromptSections.ts](../../src/constants/systemPromptSections.ts) 实现了片段级别的缓存：

```typescript
// 稳定片段：计算一次，缓存到 /clear 或 /compact
export function systemPromptSection(
  name: string,
  compute: ComputeFn,
): SystemPromptSection {
  return { name, compute, cacheBreak: false }
}

// 易变片段：每轮重新计算（会破坏 Prompt 缓存）
// 必须提供 reason 说明为何需要每轮刷新
export function DANGEROUS_uncachedSystemPromptSection(
  name: string,
  compute: ComputeFn,
  _reason: string,  // 文档化破坏缓存的原因
): SystemPromptSection {
  return { name, compute, cacheBreak: true }
}
```

**命名规范**：`DANGEROUS_` 前缀是一种工程约定，提醒开发者这个函数会破坏 Prompt 缓存，使用时需要谨慎并提供理由。

**实际使用示例**：

```typescript
// MCP 服务器会在会话中途连接/断开，必须每轮刷新
DANGEROUS_uncachedSystemPromptSection(
  'mcp_instructions',
  () => getMcpInstructionsSection(mcpClients),
  'MCP servers connect/disconnect between turns',  // 明确说明原因
)
```

---

## 16.3 身份定义的写法

```typescript
// src/constants/prompts.ts - getSimpleIntroSection()
function getSimpleIntroSection(outputStyleConfig): string {
  return `
You are an interactive agent that helps users ${
  outputStyleConfig !== null
    ? 'according to your "Output Style" below, which describes how you should respond to user queries.'
    : 'with software engineering tasks.'
} Use the instructions below and the tools available to you to assist the user.

${CYBER_RISK_INSTRUCTION}
IMPORTANT: You must NEVER generate or guess URLs for the user unless you are confident that the URLs are for helping the user with programming.`
}
```

**规范要点**：
- 身份定义简洁，一句话说明角色
- 支持 `outputStyleConfig` 动态替换角色描述（用于自定义输出风格）
- 安全指令（`CYBER_RISK_INSTRUCTION`）紧跟身份定义，确保优先级

### 安全指令的独立文件管理

```typescript
// src/constants/cyberRiskInstruction.ts
/**
 * IMPORTANT: DO NOT MODIFY THIS INSTRUCTION WITHOUT SAFEGUARDS TEAM REVIEW
 * This instruction is owned by the Safeguards team...
 */
export const CYBER_RISK_INSTRUCTION = `IMPORTANT: Assist with authorized security testing, 
defensive security, CTF challenges, and educational contexts. Refuse requests for destructive 
techniques, DoS attacks, mass targeting, supply chain compromise, or detection evasion for 
malicious purposes. Dual-use security tools (C2 frameworks, credential testing, exploit 
development) require clear authorization context: pentesting engagements, CTF competitions, 
security research, or defensive use cases.`
```

**规范要点**：安全关键指令独立成文件，文件头注释明确标注所有者（Safeguards 团队）和修改流程，防止随意改动。

---

## 16.4 行为约束的表达模式

### 模式一：正面 + 反面双重约束

```typescript
// getSimpleDoingTasksSection() 中的代码风格约束
`Don't add features, refactor code, or make "improvements" beyond what was asked.
A bug fix doesn't need surrounding code cleaned up.`

// 同时给出正面指导
`Only add comments where the logic isn't self-evident.`
```

**规范**：先说"不要做什么"，再说"什么情况下可以做"，避免绝对禁止导致的过度保守。

### 模式二：具体示例锚定抽象规则

```typescript
// 抽象规则
`When given an unclear or generic instruction, consider it in the context of 
these software engineering tasks and the current working directory.`

// 立即跟上具体示例
`For example, if the user asks you to change "methodName" to snake case, 
do not reply with just "method_name", instead find the method in the code 
and modify the code.`
```

**规范**：抽象规则后必须跟具体示例，防止模型对规则的理解产生偏差。

### 模式三：数值锚定（内部用户专用）

```typescript
// 内部用户（ant）专用的数值约束
systemPromptSection(
  'numeric_length_anchors',
  () => 'Length limits: keep text between tool calls to ≤25 words. ' +
        'Keep final responses to ≤100 words unless the task requires more detail.',
)
```

**规范**：数值锚定比定性描述（"简洁"）更有效，但需要通过 A/B 测试验证后才推广到外部用户。

### 模式四：用户类型分支

```typescript
// 内部用户（ant）获得更严格的代码注释规范
...(process.env.USER_TYPE === 'ant'
  ? [
      `Default to writing no comments. Only add one when the WHY is non-obvious...`,
      `Don't explain WHAT the code does, since well-named identifiers already do that.`,
      // @[MODEL LAUNCH]: capy v8 thoroughness counterweight — un-gate once validated
      `Before reporting a task complete, verify it actually works: run the test, 
       execute the script, check the output.`,
    ]
  : []),
```

**规范**：内部用户是新功能的试验场，通过 `process.env.USER_TYPE === 'ant'` 门控，验证后再推广。注释中标注 `@[MODEL LAUNCH]` 表示与模型发布相关的临时代码。

---

## 16.5 工具使用指导的写法

```typescript
// getUsingYourToolsSection()
const items = [
  // 核心规则：大写 CRITICAL 强调
  `Do NOT use the ${BASH_TOOL_NAME} to run commands when a relevant dedicated tool is provided. 
   Using dedicated tools allows the user to better understand and review your work. 
   This is CRITICAL to assisting the user:`,
  
  // 子条目：具体的工具替代关系
  [
    `To read files use ${FILE_READ_TOOL_NAME} instead of cat, head, tail, or sed`,
    `To edit files use ${FILE_EDIT_TOOL_NAME} instead of sed or awk`,
    `To create files use ${FILE_WRITE_TOOL_NAME} instead of cat with heredoc or echo redirection`,
    `To search for files use ${GLOB_TOOL_NAME} instead of find or ls`,
    `To search the content of files, use ${GREP_TOOL_NAME} instead of grep or rg`,
  ],
  
  // 并行执行指导
  `You can call multiple tools in a single response. If you intend to call multiple tools 
   and there are no dependencies between them, make all independent tool calls in parallel.`,
]
```

**规范要点**：
1. 工具名使用常量引用（`FILE_READ_TOOL_NAME`），避免硬编码字符串，防止工具重命名时遗漏更新
2. 替代关系明确列出（`instead of cat, head, tail, or sed`），不让模型猜测
3. 并行执行指导放在工具使用章节，与工具使用语境绑定

---

## 16.6 危险操作守则的结构

```typescript
// getActionsSection() - 完整保留原文结构
`# Executing actions with care

Carefully consider the reversibility and blast radius of actions.

Examples of the kind of risky actions that warrant user confirmation:
- Destructive operations: deleting files/branches, dropping database tables, 
  killing processes, rm -rf, overwriting uncommitted changes
- Hard-to-reverse operations: force-pushing, git reset --hard, amending published commits
- Actions visible to others or that affect shared state: pushing code, 
  creating/closing/commenting on PRs or issues, sending messages (Slack, email, GitHub)
- Uploading content to third-party web tools (diagram renderers, pastebins, gists) 
  publishes it - consider whether it could be sensitive before sending`
```

**规范要点**：
- 使用"blast radius"（爆炸半径）这一具体比喻，让模型理解风险的传播范围
- 按操作类型分类（破坏性/不可逆/可见性），而非按工具分类
- 每类给出 2-4 个具体示例，避免模型过度泛化

---

## 16.7 输出风格的双轨制

内部用户和外部用户获得截然不同的输出指导：

```typescript
function getOutputEfficiencySection(): string {
  if (process.env.USER_TYPE === 'ant') {
    // 内部用户：强调可读性和准确性，允许更长的输出
    return `# Communicating with the user
When sending user-facing text, you're writing for a person, not logging to a console.
...
Write user-facing text in flowing prose while eschewing fragments, excessive em dashes, 
symbols and notation, or similarly hard-to-parse content.
...
What's most important is the reader understanding your output without mental overhead 
or follow-ups, not how terse you are.`
  }
  
  // 外部用户：强调简洁
  return `# Output efficiency

IMPORTANT: Go straight to the point. Try the simplest approach first without going in circles.

Keep your text output brief and direct. Lead with the answer or action, not the reasoning.
Skip filler words, preamble, and unnecessary transitions.

Focus text output on:
- Decisions that need the user's input
- High-level status updates at natural milestones
- Errors or blockers that change the plan`
}
```

**规范对比**：

| 维度 | 内部用户（ant） | 外部用户 |
|------|----------------|---------|
| 核心目标 | 可读性，零心智负担 | 简洁，直达要点 |
| 长度限制 | 数值锚定（≤25/≤100 词） | 定性（"brief and direct"） |
| 格式要求 | 流畅散文，避免碎片化 | 无特殊格式要求 |
| 错误报告 | 忠实报告，不掩盖失败 | 无特殊要求 |

---

## 16.8 语气与格式规范

```typescript
// getSimpleToneAndStyleSection()
const items = [
  `Only use emojis if the user explicitly requests it.`,
  
  // 代码引用格式：文件路径 + 行号
  `When referencing specific functions or pieces of code include the pattern 
   file_path:line_number to allow the user to easily navigate to the source code location.`,
  
  // GitHub 引用格式
  `When referencing GitHub issues or pull requests, use the owner/repo#123 format 
   (e.g. anthropics/claude-code#100) so they render as clickable links.`,
  
  // 工具调用前不加冒号
  `Do not use a colon before tool calls. Your tool calls may not be shown directly 
   in the output, so text like "Let me read the file:" followed by a read tool call 
   should just be "Let me read the file." with a period.`,
]
```

**规范要点**：格式规范都有明确的**理由**（`so they render as clickable links`，`Your tool calls may not be shown directly`），让模型理解规范背后的意图，而不是盲目遵守。

---

## 16.9 压缩提示词的工程设计

[src/services/compact/prompt.ts](../../src/services/compact/prompt.ts) 展示了另一类提示词的精心设计——用于压缩对话历史的提示词。

### 防止工具调用的强制前缀

```typescript
const NO_TOOLS_PREAMBLE = `CRITICAL: Respond with TEXT ONLY. Do NOT call any tools.

- Do NOT use Read, Bash, Grep, Glob, Edit, Write, or ANY other tool.
- You already have all the context you need in the conversation above.
- Tool calls will be REJECTED and will waste your only turn — you will fail the task.
- Your entire response must be plain text: an <analysis> block followed by a <summary> block.

`
```

**工程背景**：注释解释了为何需要这个强制前缀——Sonnet 4.6 的自适应思考模式有 2.79% 的概率在压缩时尝试工具调用，导致唯一的输出机会被浪费。将此指令放在**最前面**并明确说明后果（"you will fail the task"）可以有效抑制这一行为。

同时在末尾还有一个**重复提醒**：

```typescript
const NO_TOOLS_TRAILER =
  '\n\nREMINDER: Do NOT call any tools. Respond with plain text only — ' +
  'an <analysis> block followed by a <summary> block. ' +
  'Tool calls will be rejected and you will fail the task.'
```

**规范**：对于关键约束，在提示词的**开头和结尾**各放一次，形成"三明治"结构，提高遵从率。

### 两阶段输出结构

```typescript
const DETAILED_ANALYSIS_INSTRUCTION_BASE = `Before providing your final summary, 
wrap your analysis in <analysis> tags to organize your thoughts...

1. Chronologically analyze each message and section of the conversation.
   For each section thoroughly identify:
   - The user's explicit requests and intents
   - Key decisions, technical concepts and code patterns
   - Specific details like file names, full code snippets, function signatures
   - Errors that you ran into and how you fixed them
2. Double-check for technical accuracy and completeness.`
```

**设计意图**：`<analysis>` 块是一个**草稿区**（drafting scratchpad），让模型先整理思路再输出摘要，提高摘要质量。最终 `formatCompactSummary()` 会将 `<analysis>` 块剥离，只保留 `<summary>` 内容：

```typescript
export function formatCompactSummary(summary: string): string {
  // 剥离分析块——它是提升摘要质量的草稿，写完即无用
  formattedSummary = formattedSummary.replace(
    /<analysis>[\s\S]*?<\/analysis>/,
    '',
  )
  // 提取并格式化摘要块
  const summaryMatch = formattedSummary.match(/<summary>([\s\S]*?)<\/summary>/)
  // ...
}
```

### 续写指令的精确措辞

```typescript
// 压缩后的续写指令
`Continue the conversation from where it left off without asking the user any further questions. 
Resume directly — do not acknowledge the summary, do not recap what was happening, 
do not preface with "I'll continue" or similar. 
Pick up the last task as if the break never happened.`
```

**规范**：续写指令明确列出**禁止行为**（不要确认摘要、不要回顾、不要说"我继续"），而不只是说"继续工作"。这些禁止行为都是模型在没有明确指令时的默认行为。

---

## 16.10 环境信息注入的格式

```typescript
// computeSimpleEnvInfo() - 环境信息的结构化注入
const envItems = [
  `Primary working directory: ${cwd}`,
  isWorktree
    ? `This is a git worktree — an isolated copy of the repository. 
       Run all commands from this directory. Do NOT \`cd\` to the original repository root.`
    : null,
  `Is a git repository: ${isGit}`,
  `Platform: ${env.platform}`,
  getShellInfoLine(),
  `OS Version: ${unameSR}`,
  modelDescription,
  knowledgeCutoffMessage,
  // 告知模型最新的 Claude 模型 ID，用于构建 AI 应用时的默认选择
  `The most recent Claude model family is Claude 4.5/4.6. Model IDs — 
   Opus 4.6: 'claude-opus-4-6', Sonnet 4.6: 'claude-sonnet-4-6', 
   Haiku 4.5: 'claude-haiku-4-5-20251001'. 
   When building AI applications, default to the latest and most capable Claude models.`,
]
```

**规范要点**：
- 环境信息以列表形式注入，每项一行，便于模型解析
- 特殊状态（worktree）立即跟上行为指导（"Do NOT `cd` to the original repository root"）
- 模型信息包含具体 ID，让模型在生成代码时能使用正确的模型名称

---

## 16.11 提示词版本管理规范

代码中有一套约定俗成的注释标记，用于管理与模型发布相关的提示词变更：

```typescript
// @[MODEL LAUNCH]: Update the latest frontier model.
const FRONTIER_MODEL_NAME = 'Claude Opus 4.6'

// @[MODEL LAUNCH]: Remove this section when we launch numbat.
function getOutputEfficiencySection(): string { ... }

// @[MODEL LAUNCH]: capy v8 thoroughness counterweight (PR #24302) 
// — un-gate once validated on external via A/B
`Before reporting a task complete, verify it actually works...`

// @[MODEL LAUNCH]: False-claims mitigation for Capybara v8 (29-30% FC rate vs v4's 16.7%)
`Report outcomes faithfully: if tests fail, say so...`
```

**规范**：
- `@[MODEL LAUNCH]` 标记所有需要在模型发布时更新的代码
- 注释中包含 PR 编号（`PR #24302`）便于追溯
- 内部代号（`capy`、`numbat`、`Capybara`）是模型的内部名称
- 量化指标（`29-30% FC rate vs v4's 16.7%`）说明为何需要这个修复

---

## 16.12 提示词工程总结

Claude Code 的提示词系统体现了以下核心原则：

### 1. 缓存优先设计
将系统提示分为静态区域（全局缓存）和动态区域（会话缓存），通过 `SYSTEM_PROMPT_DYNAMIC_BOUNDARY` 标记边界，最大化缓存命中率。

### 2. 规则 + 理由 + 示例三元组
每条行为规范都包含：规则本身、为何需要这条规则、具体示例。让模型理解意图，而不只是记忆规则。

### 3. 关键约束的"三明治"结构
对于必须遵守的约束（如压缩时禁止工具调用），在提示词开头和结尾各放一次，中间是正文内容。

### 4. 用户类型分支实验
通过 `process.env.USER_TYPE === 'ant'` 门控，内部用户先试验新的提示词策略，A/B 验证后再推广到外部用户。

### 5. 安全指令的独立管理
安全关键指令（`CYBER_RISK_INSTRUCTION`）独立成文件，标注所有者和修改流程，防止随意改动。

### 6. 工具名常量化
所有工具名通过常量引用（`FILE_READ_TOOL_NAME`、`BASH_TOOL_NAME`），避免硬编码字符串，确保工具重命名时提示词自动更新。

### 7. 两阶段输出模式
对于复杂的生成任务（如压缩摘要），使用 `<analysis>` + `<summary>` 两阶段结构：先让模型在草稿区整理思路，再输出最终结果，最后剥离草稿区内容。

### 8. 版本化注释规范
使用 `@[MODEL LAUNCH]` 标记与模型发布相关的临时代码，配合 PR 编号和量化指标，确保提示词随模型迭代而演进。
