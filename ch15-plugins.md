# 第 15 章：插件与技能系统

## 15.1 插件架构

Claude Code 的插件系统允许用户和组织扩展 AI 的能力：

```
插件类型:
┌─────────────────────────────────────────────────────┐
│ 内置插件（bundled/）  ← 随 Claude Code 发布          │
│ 用户插件             ← ~/.claude/plugins/            │
│ 项目插件             ← .claude/plugins/              │
│ 托管插件             ← 企业管理员分发                 │
└─────────────────────────────────────────────────────┘
```

---

## 15.2 Skill 系统

[src/skills/](../../src/skills/) 实现了技能（Skill）系统——预定义的工作流：

```typescript
// 技能定义格式
type Skill = {
  name: string
  description: string
  // 技能的系统提示
  prompt: string
  // 技能可以使用的工具
  tools?: string[]
}

// 内置技能示例
const BUILT_IN_SKILLS = [
  {
    name: 'commit',
    description: '创建 git commit',
    prompt: '分析变更，生成规范的 commit 消息，然后执行 git commit',
  },
  {
    name: 'review-pr',
    description: '审查 Pull Request',
    prompt: '分析 PR 的变更，提供详细的代码审查意见',
  },
];
```

### 技能调用

用户通过斜杠命令调用技能：

```
/commit          → 触发 commit 技能
/review-pr 123   → 触发 review-pr 技能，参数为 PR #123
```

---

## 15.3 Hooks 机制

[src/utils/hooks/](../../src/utils/hooks/) 实现了 Hooks 系统，允许在工具执行的各个阶段插入自定义逻辑：

### 25 个 Hook 事件

```typescript
type HookEvent =
  // 会话生命周期
  | 'session_start'
  | 'session_end'
  
  // 工具执行
  | 'pre_tool_use'
  | 'post_tool_use'
  
  // Agent 生命周期
  | 'pre_agent_start'
  | 'post_agent_end'
  
  // 用户交互
  | 'pre_user_input'
  | 'post_user_input'
  
  // 对话管理
  | 'pre_compact'
  | 'post_compact'
  
  // ...更多事件
```

### Hook 配置

```json
// settings.json
{
  "hooks": {
    "pre_tool_use": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "echo 'About to run Bash command'"
          }
        ]
      }
    ],
    "post_tool_use": [
      {
        "matcher": "FileEdit",
        "hooks": [
          {
            "type": "command",
            "command": "prettier --write $TOOL_INPUT_FILE_PATH"
          }
        ]
      }
    ]
  }
}
```

### Hook 执行流程

```
工具调用请求
    │
    ▼
执行 pre_tool_use hooks
    │
    ├─ 信任检查（交互式 vs SDK 模式）
    ├─ 过滤匹配的 hooks
    ├─ 并行执行所有 hooks
    └─ 聚合结果（continue/block/approve）
    │
    ▼
执行工具
    │
    ▼
执行 post_tool_use hooks
```

---

## 15.4 自定义命令

[src/commands/](../../src/commands/) 包含 103 个斜杠命令，用户也可以自定义：

```typescript
// 自定义命令格式（.claude/commands/my-command.md）
// ---
// description: 我的自定义命令
// ---
// 
// 执行以下操作：
// 1. 读取当前文件
// 2. 分析代码质量
// 3. 生成改进建议
```

---

## 15.5 插件生命周期

```typescript
// src/utils/plugins/pluginLoader.ts
export async function loadAllPlugins(): Promise<LoadedPlugin[]> {
  const plugins: LoadedPlugin[] = [];
  
  // 1. 加载内置插件
  const bundled = await loadBundledPlugins();
  plugins.push(...bundled);
  
  // 2. 加载用户插件
  const userPlugins = await loadPluginsFromDir('~/.claude/plugins');
  plugins.push(...userPlugins);
  
  // 3. 加载项目插件
  const projectPlugins = await loadPluginsFromDir('.claude/plugins');
  plugins.push(...projectPlugins);
  
  return plugins;
}
```

---

## 小结

Claude Code 的插件和技能系统提供了强大的扩展能力：
- **Skill 系统**：预定义工作流，通过斜杠命令触发
- **Hooks 机制**：在工具执行各阶段插入自定义逻辑
- **自定义命令**：用 Markdown 文件定义新的斜杠命令
- **插件系统**：支持用户、项目、企业级别的功能扩展
