# 第 16 章：状态管理

## 16.1 状态管理架构

Claude Code 使用了一个简洁但有效的状态管理方案，类似于 Redux 但更轻量：

```
createStore()  ← 通用 Store 工厂
    │
    ▼
AppState Store  ← 主应用状态
    │
    ├─ UI 状态（消息列表、加载状态等）
    ├─ 权限上下文
    ├─ 模型配置
    ├─ 任务状态
    └─ 工具进度
```

---

## 16.2 Store 实现

[src/state/store.ts](../../src/state/store.ts) 实现了一个极简的响应式 Store：

```typescript
export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}

export function createStore<T>(
  initialState: T,
  onChange?: OnChange<T>,
): Store<T> {
  let state = initialState;
  const listeners = new Set<Listener>();

  return {
    getState: () => state,

    setState: (updater: (prev: T) => T) => {
      const prev = state;
      const next = updater(prev);
      
      // 引用相等检查，避免不必要的更新
      if (Object.is(next, prev)) return;
      
      state = next;
      onChange?.({ newState: next, oldState: prev });
      
      // 通知所有订阅者
      for (const listener of listeners) listener();
    },

    subscribe: (listener: Listener) => {
      listeners.add(listener);
      return () => listeners.delete(listener);  // 返回取消订阅函数
    },
  };
}
```

这个实现只有 30 行，但包含了状态管理的核心功能：
- **不可变更新**：通过 updater 函数确保状态不可变
- **引用相等优化**：`Object.is` 检查避免无效更新
- **订阅/取消订阅**：标准的观察者模式

---

## 16.3 AppState 结构

[src/state/AppStateStore.ts](../../src/state/AppStateStore.ts) 定义了完整的应用状态：

```typescript
export type AppState = DeepImmutable<{
  // 设置
  settings: SettingsJson
  verbose: boolean
  mainLoopModel: ModelSetting
  
  // UI 状态
  statusLineText: string | undefined
  expandedView: 'none' | 'tasks' | 'teammates'
  isBriefOnly: boolean
  
  // 消息
  messages: Message[]
  
  // 权限
  toolPermissionContext: ToolPermissionContext
  
  // 工具进度
  inProgressToolUseIDs: Set<string>
  hasInterruptibleToolInProgress: boolean
  
  // 任务
  tasks: TaskState[]
  
  // 推测执行
  speculationState: SpeculationState
  
  // MCP
  mcpClients: MCPServerConnection[]
  mcpResources: Record<string, ServerResource[]>
  
  // 插件
  loadedPlugins: LoadedPlugin[]
  pluginErrors: PluginError[]
  
  // 文件历史
  fileHistoryState: FileHistoryState
  
  // 归因状态（git commit 追踪）
  attributionState: AttributionState
  
  // ...更多字段
}>
```

`DeepImmutable<T>` 类型确保状态对象在 TypeScript 层面是只读的，防止意外修改。

---

## 16.4 状态变更检测

[src/state/onChangeAppState.ts](../../src/state/onChangeAppState.ts) 处理状态变更的副作用：

```typescript
export function onChangeAppState(
  { newState, oldState }: { newState: AppState; oldState: AppState }
): void {
  // 检测模型变更
  if (newState.mainLoopModel !== oldState.mainLoopModel) {
    logEvent('tengu_model_changed', { model: newState.mainLoopModel });
  }
  
  // 检测权限模式变更
  if (newState.toolPermissionContext.mode !== oldState.toolPermissionContext.mode) {
    saveMode(newState.toolPermissionContext.mode);
  }
  
  // 检测设置变更
  if (newState.settings !== oldState.settings) {
    applySettingsChanges(newState.settings);
  }
}
```

---

## 16.5 React Hooks 集成

[src/hooks/](../../src/hooks/) 目录包含 87 个自定义 React Hooks，将 Store 状态桥接到 React 组件：

```typescript
// src/hooks/useAppState.ts
export function useAppState(): AppState {
  const store = useContext(AppStateContext);
  const [state, setState] = useState(store.getState());
  
  useEffect(() => {
    // 订阅 Store 变更
    const unsubscribe = store.subscribe(() => {
      setState(store.getState());
    });
    return unsubscribe;
  }, [store]);
  
  return state;
}

// 使用示例
function StatusLine() {
  const { mainLoopModel, totalCostUSD } = useAppState();
  return <Text>{mainLoopModel} | ${totalCostUSD.toFixed(4)}</Text>;
}
```

---

## 16.6 全局状态 vs 应用状态

Claude Code 有两套状态系统：

| 特性 | bootstrap/state.ts | AppStateStore.ts |
|------|-------------------|-----------------|
| 作用域 | 全局单例 | React 组件树 |
| 访问方式 | 直接函数调用 | React Context + Hooks |
| 持久化 | 进程生命周期 | 会话生命周期 |
| 用途 | 会话 ID、成本追踪、遥测 | UI 状态、消息列表、权限 |

---

## 小结

Claude Code 的状态管理采用了"够用就好"的设计哲学：
- 极简的 Store 实现（30 行）满足了所有需求
- `DeepImmutable` 类型提供编译时安全保证
- 两套状态系统分别处理全局配置和 UI 状态
- 87 个自定义 Hooks 提供了丰富的状态访问接口
