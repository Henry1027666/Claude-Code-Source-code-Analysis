# 第 13 章：内存系统

## 13.1 七层记忆架构

Claude Code 实现了一套七层记忆系统，覆盖从全局系统指令到会话级临时笔记的不同作用域：

```
优先级（高→低）          层级              存储位置
─────────────────────────────────────────────────────────
  最高  ┌─ Layer 1: Managed Memory    /etc/claude-code/CLAUDE.md
        ├─ Layer 2: User Memory       ~/.claude/CLAUDE.md
        ├─ Layer 3: Project Memory    <project>/CLAUDE.md
        ├─ Layer 4: Local Memory      <project>/CLAUDE.local.md
        ├─ Layer 5: Auto Memory       ~/.claude/projects/<slug>/memory/
        ├─ Layer 6: Team Memory       ~/.claude/projects/<slug>/memory/team/
  最低  └─ Layer 7: Session Memory    ~/.claude/projects/<slug>/memory/session-<id>.md
─────────────────────────────────────────────────────────
```

所有层在启动时通过 [src/context.ts](../../src/context.ts) 的 `getUserContext()` 统一加载，注入系统提示词。

---

## 13.2 Layer 1-4：CLAUDE.md 文件系统

[src/utils/claudemd.ts](../../src/utils/claudemd.ts)（1480 行）处理前四层记忆的发现与加载。

### 目录树向上遍历算法

```typescript
// getMemoryFiles() 的核心逻辑
function getMemoryFiles(cwd: string): MemoryFile[] {
  const files: MemoryFile[] = [];
  let dir = cwd;
  
  // 从当前目录向上遍历到根目录
  while (dir !== path.dirname(dir)) {
    // 检查每一层的 CLAUDE.md 文件
    const candidates = [
      path.join(dir, 'CLAUDE.md'),
      path.join(dir, '.claude', 'CLAUDE.md'),
      ...glob.sync(path.join(dir, '.claude', 'rules', '*.md')),
    ];
    
    for (const candidate of candidates) {
      if (fs.existsSync(candidate)) {
        files.push({ path: candidate, priority: distanceFromCwd(dir, cwd) });
      }
    }
    
    dir = path.dirname(dir);
  }
  
  // 越靠近 CWD 的文件优先级越高
  return files.sort((a, b) => a.priority - b.priority);
}
```

### @include 引用处理

CLAUDE.md 文件支持 `@include` 引用其他文件：

```markdown
<!-- CLAUDE.md -->
# 项目规范

@./docs/coding-standards.md
@./docs/api-guidelines.md
```

```typescript
// processMemoryFile() 处理 @include
function processMemoryFile(filePath: string, depth: number): string {
  if (depth > MAX_INCLUDE_DEPTH) {  // MAX_INCLUDE_DEPTH = 5
    return '<!-- 超过最大引用深度 -->';
  }
  
  const content = fs.readFileSync(filePath, 'utf8');
  
  // 解析 @path 引用
  return content.replace(/@([\w./\-]+)/g, (match, includePath) => {
    const resolvedPath = path.resolve(path.dirname(filePath), includePath);
    
    // 循环引用检测
    if (visitedPaths.has(resolvedPath)) {
      return '<!-- 循环引用 -->';
    }
    
    // 外部 include 需要用户审批
    if (isExternalPath(resolvedPath)) {
      if (!hasClaudeMdExternalIncludesApproved()) {
        return '<!-- 外部引用需要审批 -->';
      }
    }
    
    return processMemoryFile(resolvedPath, depth + 1);
  });
}
```

---

## 13.3 Layer 5：Auto Memory（自动记忆）

[src/memdir/memdir.ts](../../src/memdir/memdir.ts) 实现了跨会话持久化的自动记忆系统。

### 四类记忆分类

```typescript
type MemoryType = 'user' | 'feedback' | 'project' | 'reference';

// user: 用户角色、偏好、知识背景
// feedback: 用户对 AI 行为的纠正与确认
// project: 项目目标、决策、里程碑
// reference: 外部系统资源指针
```

### MEMORY.md 格式

```markdown
---
name: user_role
description: 用户是高级 Go 工程师，首次接触 React
type: user
---

用户有 10 年 Go 经验，但这是他第一次接触这个项目的 React 前端。
解释前端概念时，用 Go 的类比来帮助理解。

---
name: feedback_testing
description: 集成测试必须使用真实数据库
type: feedback
---

不要在测试中 mock 数据库。
**Why:** 上季度 mock 测试通过但生产迁移失败。
**How to apply:** 所有数据库测试必须连接真实测试数据库。
```

### MEMORY.md 截断保护

```typescript
// src/memdir/memdir.ts
function truncateEntrypointContent(content: string): string {
  const MAX_LINES = 200;
  const MAX_BYTES = 25_000;
  
  const lines = content.split('\n');
  
  if (lines.length <= MAX_LINES && Buffer.byteLength(content) <= MAX_BYTES) {
    return content;
  }
  
  // 截断并追加警告
  const truncated = lines.slice(0, MAX_LINES).join('\n');
  return truncated + '\n\n<!-- MEMORY.md 已截断，请清理旧条目 -->';
}
```

---

## 13.4 相关记忆检索

[src/memdir/findRelevantMemories.ts](../../src/memdir/findRelevantMemories.ts) 使用 AI 语义检索相关记忆：

```typescript
async function findRelevantMemories(
  currentMessages: Message[],
  alreadySurfaced: Set<string>,
): Promise<MemoryFile[]> {
  // 1. 扫描所有记忆文件
  const allMemories = await scanMemoryFiles();
  
  // 2. 过滤已展示过的
  const candidates = allMemories.filter(m => !alreadySurfaced.has(m.path));
  
  // 3. 用 Sonnet 语义选择最多 5 条相关记忆
  const relevant = await sideQuery(
    buildMemorySelectionPrompt(currentMessages, candidates),
    { model: 'claude-sonnet-4-6', maxTokens: 1000 }
  );
  
  return parseSelectedMemories(relevant);
}
```

---

## 13.5 Layer 7：Session Memory（会话记忆）

[src/services/SessionMemory/sessionMemory.ts](../../src/services/SessionMemory/sessionMemory.ts) 在会话过程中自动提取关键信息：

### 提取触发条件

```typescript
// 三个阈值，任一满足即触发
const THRESHOLDS = {
  minimumMessageTokensToInit: 2000,    // 首次提取的最低 token 数
  minimumTokensBetweenUpdate: 1000,    // 两次提取间的最低 token 增量
  toolCallsBetweenUpdates: 10,         // 工具调用次数阈值
};
```

### 提取流程

```
Post-sampling Hook 触发
    │
    ▼
runForkedAgent()  →  隔离上下文的子 Agent
    │
    ▼
buildSessionMemoryUpdatePrompt()  →  构建提取提示词
    │
    ▼
FileEditTool（仅限操作该 session 文件）
    │
    ▼
updateLastSummarizedMessageIdIfSafe()  →  更新追踪状态
```

**关键设计**：
- 非阻塞后台执行，不影响主对话
- 超时：15 秒；过期：1 分钟
- `/summary` 命令可手动触发

---

## 13.6 统一加载流程

```
getUserContext()  [src/context.ts]
    │
    ▼
getMemoryFiles()  [src/utils/claudemd.ts]  ← memoized
    ├─ Layer 1: Managed  (/etc/claude-code/CLAUDE.md)
    ├─ Layer 2: User     (~/.claude/CLAUDE.md)
    ├─ Layer 3: Project  (CLAUDE.md 向上遍历)
    ├─ Layer 4: Local    (CLAUDE.local.md)
    ├─ Layer 5: AutoMem  (MEMORY.md)
    └─ Layer 6: TeamMem  (team/MEMORY.md)
    │
    ▼
格式化为系统提示词文本
    │
    ▼
注入 system prompt

Layer 7: Session Memory（独立流程）
    │
    ▼
initSessionMemory()  →  注册 post-sampling hook
    │
    ▼
每次采样后异步提取，独立于主流程
```

---

## 小结

Claude Code 的七层记忆系统是一个精心设计的分层架构：
- **Layer 1-4**：基于文件系统的静态记忆，适合团队共享规范
- **Layer 5-6**：跨会话持久化的动态记忆，适合个人和团队偏好
- **Layer 7**：会话内自动提取，适合临时上下文保存

这个系统使 Claude Code 能够在不同会话间保持上下文连续性，同时支持团队级别���知识共享。
