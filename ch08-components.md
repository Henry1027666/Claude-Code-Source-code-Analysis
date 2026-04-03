# 第 8 章：UI 组件系统

## 8.1 原子设计系统

Claude Code 的 UI 组件遵循原子设计（Atomic Design）原则，从最小的原子组件到完整的页面：

```
Atoms（原子）
  └─ ink/ 基础组件（Text, Box, Spinner）
  
Molecules（分子）
  └─ components/ 复合组件（Message, StatusLine）
  
Organisms（有机体）
  └─ components/ 功能组件（PromptInput, MessageList）
  
Templates（模板）
  └─ commands/ 斜杠命令
  
Pages（页面）
  └─ screens/ 完整页面（REPL, Doctor）
```

---

## 8.2 核心组件：REPL.tsx

[src/screens/REPL.tsx](../../src/screens/REPL.tsx)（895KB）是主界面组件，包含：

```typescript
function REPL({ config }: REPLProps) {
  // 状态管理
  const [appState, setAppState] = useAppState();
  const messages = appState.messages;
  
  // 消息提交处理
  const handleSubmit = useCallback(async (input: string) => {
    // 1. 添加用户消息到列表
    setAppState(prev => ({
      ...prev,
      messages: [...prev.messages, createUserMessage(input)],
    }));
    
    // 2. 启动查询引擎
    for await (const event of queryEngine.submitMessage(input)) {
      // 3. 实时更新 UI
      if (event.type === 'assistant') {
        setAppState(prev => ({
          ...prev,
          messages: [...prev.messages, event],
        }));
      }
    }
  }, [queryEngine]);
  
  return (
    <Box flexDirection="column" height="100%">
      <Messages messages={messages} />
      <StatusLine model={appState.mainLoopModel} />
      <PromptInput onSubmit={handleSubmit} />
    </Box>
  );
}
```

---

## 8.3 PromptInput：输入系统

[src/components/PromptInput/](../../src/components/PromptInput/)（23 个文件）实现了功能丰富的输入组件：

```typescript
// 支持的功能：
// - 多行输入（Shift+Enter 换行）
// - 历史记录（上下箭头）
// - 自动补全（Tab）
// - Vim 模式（可选）
// - 粘贴检测（大段文本自动处理）
// - 文件拖放
// - 图片粘贴

function PromptInput({ onSubmit, placeholder }: PromptInputProps) {
  const [value, setValue] = useState('');
  const [history, setHistory] = useInputHistory();
  const isVimMode = useVimMode();
  
  const handleKeyPress = useCallback((key: Keypress) => {
    if (isVimMode) {
      return handleVimKeyPress(key, value, setValue);
    }
    
    switch (key.key) {
      case 'return':
        if (!key.shift) {
          onSubmit(value);
          setValue('');
        } else {
          setValue(v => v + '\n');
        }
        break;
      case 'upArrow':
        setValue(history.previous());
        break;
      // ...
    }
  }, [value, isVimMode, history]);
  
  return (
    <Box>
      <Text color="cyan">{'> '}</Text>
      <TextInput value={value} onChange={setValue} onKeyPress={handleKeyPress} />
    </Box>
  );
}
```

---

## 8.4 Vim 模式实现

[src/vim/](../../src/vim/) 实现了完整的 Vim 编辑模式：

```typescript
// Vim 模式状态机
type VimMode = 'normal' | 'insert' | 'visual' | 'command';

// 支持的 Vim 操作：
// 移动：h/j/k/l, w/b/e, 0/$/^, gg/G
// 编辑：i/a/o/O, x/X, d/c/y/p
// 文本对象：iw/aw, i"/a", i(/a(
// 命令：:w, :q, /search, n/N
```

---

## 8.5 消息渲染

[src/components/messages/](../../src/components/messages/)（36 个文件）处理不同类型消息的渲染：

```typescript
// 消息类型 → 渲染组件映射
const MESSAGE_RENDERERS = {
  'user': UserMessage,
  'assistant': AssistantMessage,
  'tool_use': ToolUseMessage,
  'tool_result': ToolResultMessage,
  'system': SystemMessage,
  'error': ErrorMessage,
};

// Markdown 渲染
function AssistantMessage({ content }: { content: string }) {
  const parsed = marked.parse(content);
  return <MarkdownRenderer content={parsed} />;
}
```

---

## 8.6 主题系统

```typescript
// src/utils/theme.ts
export type Theme = {
  text: string
  secondaryText: string
  background: string
  permission: string
  error: string
  warning: string
  success: string
  // ...
};

// 内置主题
const THEMES = {
  dark: { text: 'white', background: 'black', ... },
  light: { text: 'black', background: 'white', ... },
  solarized: { ... },
};
```

---

## 小结

Claude Code 的 UI 系统将 Web 开发的最佳实践带入了终端：
- 原子设计确保组件可复用性
- React 组件模型简化了复杂 UI 的开发
- Vim 模式满足了高级用户的需求
- 主题系统提供了个性化体验
