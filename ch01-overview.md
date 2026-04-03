# 第 1 章：项目概述与架构全景

## 1.1 Claude Code 是什么

Claude Code 是 Anthropic 官方出品的 AI 编程助手 CLI 工具，版本号 `2.1.88`，以 npm 包 `@anthropic-ai/claude-code` 发布。它不是一个简单的命令行包装器，而是一个完整的 AI 代理系统——能够理解代码库、编辑文件、执行终端命令，并自主完成复杂的工程工作流。

```
npm install -g @anthropic-ai/claude-code
claude  # 启动交互式 REPL
```

从用户视角看，Claude Code 是一个终端程序。从工程视角看，它是一个包含以下核心能力的复杂系统：

- **AI 对话引擎**：与 Claude API 进行多轮对话，管理上下文和 Token 预算
- **工具执行框架**：45 个内置工具，覆盖文件操作、代码搜索、Shell 执行、Web 访问等
- **终端 UI 系统**：基于 React + 自定义 Ink 渲染器的富文本终端界面
- **权限安全系统**：三层权限模型，保护用户系统安全
- **外部集成层**：VS Code Bridge、MCP 协议、多云 AI 平台支持

---

## 1.2 整体架构图

```
┌─────────────────────────────────────────────────────────────┐
│                        用户界面层                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐  │
│  │  终端 REPL   │  │  VS Code IDE │  │  Web (claude.ai) │  │
│  └──────┬───────┘  └──────┬───────┘  └────────┬─────────┘  │
│         │                 │ Bridge              │ Remote     │
└─────────┼─────────────────┼─────────────────────┼───────────┘
          │                 │                     │
┌─────────▼─────────────────▼─────────────────────▼───────────┐
│                       核心引擎层                              │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │                    QueryEngine                          │ │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │ │
│  │  │  Agent Loop  │  │ Token Budget │  │  Tool Router │  │ │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │ │
│  └─────────────────────────────────────────────────────────┘ │
│                                                               │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │  Permission  │  │  State Mgmt  │  │  Command Router  │   │
│  │   System     │  │  (AppState)  │  │  (103 commands)  │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │
└───────────────────────────────────────────────────────────────┘
          │                 │                     │
┌─────────▼─────────────────▼─────────────────────▼───────────┐
│                       工具执行层                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────────┐   │
│  │ 文件工具  │ │ Shell工具│ │ Web工具  │ │  Agent工具   │   │
│  │Read/Write│ │Bash/REPL │ │Fetch/    │ │  AgentTool   │   │
│  │Edit/Glob │ │PowerShell│ │Search    │ │  SkillTool   │   │
│  └──────────┘ └──────────┘ └──────────┘ └──────────────┘   │
└───────────────────────────────────────────────────────────────┘
          │                 │                     │
┌─────────▼─────────────────▼─────────────────────▼───────────┐
│                       外部服务层                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────┐   │
│  │  Claude API  │  │  MCP Servers │  │  Cloud Platforms │   │
│  │  (Anthropic) │  │  (External)  │  │  Bedrock/Vertex  │   │
│  └──────────────┘  └──────────────┘  └──────────────────┘   │
└───────────────────────────────────────────────────────────────┘
```

### 数据流：一次完整的用户请求

```
用户输入 "帮我修复这个 bug"
    │
    ▼
PromptInput 组件捕获输入
    │
    ▼
REPL.tsx 处理提交事件
    │
    ▼
QueryEngine.ts 开始 Agent 循环
    │
    ├─► 构建系统提示 (getSystemPrompt)
    ├─► 调用 Claude API (claude.ts)
    │       │
    │       ▼
    │   Claude 返回工具调用请求
    │       │
    ├─► 权限检查 (Permission System)
    │       │
    ├─► 工具执行 (e.g., BashTool, FileReadTool)
    │       │
    ├─► 结果返回给 Claude API
    │       │
    │   Claude 返回最终文本响应
    │
    ▼
Message 组件渲染到终端
```

---

## 1.3 技术栈选型分析

### 核心语言：TypeScript

整个项目由 **1,884 个 TypeScript/TSX 文件**构成。选择 TypeScript 的原因显而易见：

- 复杂的工具系统需要严格的类型约束
- AI API 响应结构需要类型安全的解析
- 大型团队协作需要类型文档化

### 运行时：Node.js + Bun

```json
// package.json
{
  "engines": { "node": ">=18.0.0" },
  "type": "module"
}
```

- **Node.js 18+**：运行时环境，利用原生 ESM 支持
- **Bun**：构建工具（非运行时），用于打包和依赖管理
  - `bun:bundle` 提供编译时 `feature()` 函数，实现死代码消除
  - `bun.lock` 替代 `package-lock.json`

### 终端 UI：React + 自定义 Ink

这是最有趣的技术选型。Claude Code 没有使用传统的终端 UI 库（如 blessed、inquirer），而是：

1. 使用 **React** 作为组件框架
2. 使用自定义的 **Ink 渲染器**（`src/ink/`）将 React 虚拟 DOM 渲染为终端输出
3. 集成 **Yoga 布局引擎**（Facebook 的 Flexbox 实现）计算终端布局

这意味着开发者可以用熟悉的 React 模式编写终端 UI：

```tsx
// src/components/Message.tsx（示意）
function Message({ content, role }) {
  return (
    <Box flexDirection="column">
      <Text color={role === 'assistant' ? 'cyan' : 'white'}>
        {content}
      </Text>
    </Box>
  );
}
```

### AI 集成：多云平台支持

```
Anthropic SDK  ──► Claude API (直连)
Bedrock SDK    ──► AWS Bedrock (企业)
Vertex SDK     ──► Google Cloud Vertex AI (企业)
Foundry SDK    ──► Anthropic Foundry (内部)
```

### 外部协议：MCP

Model Context Protocol (MCP) 是 Anthropic 推出的开放协议，允许外部工具服务器为 Claude 提供额外工具。Claude Code 实现了完整的 MCP 客户端。

---

## 1.4 构建与分发机制

### 单文件打包策略

Claude Code 采用了一个极其简洁的分发策略：**将所有代码打包成单个 `cli.js` 文件（13MB）**。

```
src/ (1,884 TypeScript 文件)
    │
    ▼ Bun 编译 + 打包
    │
cli.js (13MB, 包含所有依赖)
cli.js.map (Source Map)
```

优点：
- 用户安装后无需额外依赖解析
- 启动速度快（无模块解析开销）
- 版本一致性保证

唯一的例外是平台相关的原生模块（Sharp 图像处理库），通过 `optionalDependencies` 按平台安装：

```json
"optionalDependencies": {
  "@img/sharp-darwin-arm64": "^0.34.2",
  "@img/sharp-darwin-x64": "^0.34.2",
  "@img/sharp-linux-arm64": "^0.34.2",
  // ...8 个平台变体
}
```

### 编译时特性开关

Bun 的 `feature()` 函数实现了编译时死代码消除（DCE）：

```typescript
// src/entrypoints/cli.tsx:21
if (feature('ABLATION_BASELINE') && process.env.CLAUDE_CODE_ABLATION_BASELINE) {
  // 这段代码在外部构建中被完全消除
}

// src/main.tsx:76
const coordinatorModeModule = feature('COORDINATOR_MODE') 
  ? require('./coordinator/coordinatorMode.js') 
  : null;
```

这允许 Anthropic 在内部构建中包含实验性功能，而在公开发布版本中完全移除这些代码。

### 发布保护

```json
"scripts": {
  "prepare": "node -e \"if (!process.env.AUTHORIZED) { 
    console.error('ERROR: Direct publishing is not allowed.'); 
    process.exit(1); 
  }\""
}
```

直接 `npm publish` 会被阻止，必须通过授权的发布工作流。

---

## 1.5 代码规模与组织

### 目录结构概览

```
claude-code-source-main/
├── cli.js                 # 13MB 打包产物（分发用）
├── cli.js.map             # Source Map
├── package.json           # 包元数据
├── bun.lock               # 依赖锁定
├── sdk-tools.d.ts         # SDK 类型定义
├── src/                   # 源码（1,884 文件）
│   ├── main.tsx           # 主入口
│   ├── QueryEngine.ts     # 核心查询引擎
│   ├── Tool.ts            # 工具类型定义
│   ├── tools.ts           # 工具注册
│   ├── commands.ts        # 命令注册
│   ├── entrypoints/       # SDK 入口点
│   ├── bootstrap/         # 启动初始化
│   ├── tools/             # 45 个工具实现
│   ├── commands/          # 103 个斜杠命令
│   ├── components/        # 146 个 UI 组件
│   ├── screens/           # 全页面 UI
│   ├── hooks/             # 87 个 React Hooks
│   ├── services/          # 外部服务集成
│   │   ├── api/           # Claude API 客户端
│   │   ├── mcp/           # MCP 协议
│   │   ├── analytics/     # 遥测分析
│   │   └── ...
│   ├── utils/             # 100+ 工具函数
│   ├── ink/               # 自定义终端渲染器
│   ├── bridge/            # VS Code 集成
│   ├── remote/            # 远程会话
│   ├── coordinator/       # 多 Agent 协调
│   └── ...
├── doc/                   # 技术文档
└── vendor/                # 原生二进制（ripgrep 等）
```

### 模块规模统计

| 模块 | 文件数 | 说明 |
|------|--------|------|
| components/ | 146 | React UI 组件 |
| tools/ | 45 | 工具实现 |
| commands/ | 103 | 斜杠命令 |
| hooks/ | 87 | React Hooks |
| services/ | 38 子目录 | 外部服务 |
| utils/ | 100+ | 工具函数 |
| ink/ | 50+ | 渲染引擎 |

### 关键文件大小（反映复杂度）

| 文件 | 大小 | 说明 |
|------|------|------|
| `cli.js` | 13MB | 完整打包产物 |
| `screens/REPL.tsx` | 895KB | 主 REPL 界面 |
| `services/api/claude.ts` | 125KB | API 客户端 |
| `bridge/bridgeMain.ts` | 115KB | VS Code Bridge |
| `services/mcp/client.ts` | 119KB | MCP 客户端 |
| `cli/print.ts` | 212KB | 输出格式化 |

这些文件的体积揭示了系统的复杂度重心：**REPL 交互、API 通信、IDE 集成、MCP 协议**是最复杂的四个子系统。

---

## 1.6 五个核心概念

理解 Claude Code 架构，需要先掌握五个核心概念：

### 1. QueryEngine（查询引擎）
负责管理整个 AI 对话循环。它接收用户输入，调用 Claude API，处理工具调用，直到 AI 返回最终响应。这是整个系统的"大脑"。

### 2. Tool（工具）
AI 可以执行的具体操作。每个工具有明确的输入 Schema（Zod 验证）、执行逻辑和结果格式。工具是 AI 与真实世界交互的唯一通道。

### 3. Permission（权限）
安全护栏系统。每次工具执行前，权限系统决定是自动允许、询问用户还是拒绝。这是 Claude Code 安全性的核心保障。

### 4. Command（命令）
用户可以直接调用的斜杠命令（如 `/commit`、`/review`）。命令是预定义的工作流，通常会触发一系列工具调用。

### 5. Agent（代理）
子助手，可以并行处理复杂任务。主 Agent 可以创建子 Agent，形成多 Agent 协作网络。

---

## 小结

Claude Code 是一个架构精良的生产级 AI 工具，其设计体现了几个关键工程决策：

1. **单文件分发**：极简的用户安装体验
2. **React 终端 UI**：将 Web 开发范式引入终端
3. **编译时特性开关**：内外部构建差异化
4. **多云平台抽象**：不绑定单一 AI 提供商
5. **MCP 协议**：开放的工具扩展生态

下一章，我们将深入分析启动流程——从用户输入 `claude` 命令到 REPL 界面出现，这 200ms 内发生了什么。
