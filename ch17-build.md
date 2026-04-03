# 第 17 章：构建系统与性能优化

## 17.1 Bun 构建流程

Claude Code 使用 Bun 作为构建工具，将 1,884 个 TypeScript 文件打包成单个 `cli.js`：

```
src/ (TypeScript 源码)
    │
    ▼ bun build
    │
    ├─ TypeScript 编译（类型检查 + 转译）
    ├─ 模块打包（所有依赖内联）
    ├─ 死代码消除（feature() 门控）
    ├─ 宏替换（MACRO.VERSION 等）
    └─ Source Map 生成
    │
    ▼
cli.js (13MB) + cli.js.map
```

---

## 17.2 编译时特性开关

Bun 的 `feature()` 函数实现了编译时死代码消除（DCE）：

```typescript
// 内部构建：feature('COORDINATOR_MODE') = true
// 外部构建：feature('COORDINATOR_MODE') = false（整个 if 块被消除）

const coordinatorModeModule = feature('COORDINATOR_MODE')
  ? require('./coordinator/coordinatorMode.js')
  : null;

// 外部构建的 cli.js 中，这段代码完全不存在
```

这允许 Anthropic 在内部构建中包含实验性功能，而公开版本中完全移除。

---

## 17.3 Tree Shaking 分析

[doc/god-module-and-tree-shaking-analysis.md](../god-module-and-tree-shaking-analysis.md) 分析了模块耦合情况：

### 高耦合模块（God Modules）

| 文件 | Import 数量 | 说明 |
|------|------------|------|
| `main.tsx` | 164 | 主入口，必然高耦合 |
| `commands.ts` | 105 | 命令注册中心 |
| `REPL.tsx` | 80+ | 主 UI 组件 |
| `claude.ts` | 77 | API 客户端 |
| `QueryEngine.ts` | 51 | 查询引擎 |

### 低耦合模块（原子模块）

- 类型定义层（零耦合）
- 单个工具实现
- 工具函数
- 服务子模块

---

## 17.4 启动性能优化

### 并行 I/O 优化

```typescript
// main.tsx 启动时立即触发（与模块加载并行）
startMdmRawRead();      // MDM 策略读取（~50ms）
startKeychainPrefetch(); // Keychain 读取（~65ms）

// 这两个操作与后续 ~135ms 的模块加载并行执行
// 节省约 115ms 启动时间
```

### 懒加载重型依赖

```typescript
// OpenTelemetry (~400KB) 延迟加载
void import('../services/analytics/firstPartyEventLogger.js');

// gRPC (~700KB) 在 instrumentation.ts 内部进一步懒加载
// React/Ink 在确认需要 UI 后才加载
```

### 快速路径

```typescript
// --version 零模块加载
if (args[0] === '--version') {
  console.log(`${MACRO.VERSION} (Claude Code)`);
  return;  // 立即退出，不加载任何模块
}
```

---

## 17.5 原生模块集成

Claude Code 使用了多个原生模块：

| 模块 | 用途 | 平台 |
|------|------|------|
| `@img/sharp-*` | 图像处理（压缩截图） | 多平台 |
| `audio-capture.node` | 语音输入 | macOS |
| `image-processor` | 图像预处理 | 多平台 |
| `modifiers-napi` | 键盘修饰键检测 | 多平台 |
| `url-handler` | URL 协议处理 | macOS |
| Ripgrep | 快速文件搜索 | 多平台 |

原生模块通过 `optionalDependencies` 按平台安装：

```json
"optionalDependencies": {
  "@img/sharp-darwin-arm64": "^0.34.2",
  "@img/sharp-darwin-x64": "^0.34.2",
  "@img/sharp-linux-arm64": "^0.34.2",
  // ...8 个平台变体
}
```

---

## 17.6 性能监控

```typescript
// 启动性能分析（CLAUDE_CODE_PROFILE_STARTUP=1）
const PHASE_DEFINITIONS = {
  import_time:    ['cli_entry', 'main_tsx_imports_loaded'],
  init_time:      ['init_function_start', 'init_function_end'],
  settings_time:  ['eagerLoadSettings_start', 'eagerLoadSettings_end'],
  total_time:     ['cli_entry', 'main_after_run'],
};

// 输出示例：
// STARTUP PROFILING REPORT
// ========================
// 0.0ms  +0.0ms  profiler_initialized
// 1.2ms  +1.2ms  cli_entry
// 5.8ms  +4.6ms  main_tsx_entry
// 140.3ms +134.5ms main_tsx_imports_loaded  ← 模块加载
// 145.1ms +4.8ms  init_function_start
// 180.2ms +35.1ms init_function_end         ← 初始化
// Total startup time: 250ms
```

---

## 小结

Claude Code 的构建系统体现了几个关键优化：
1. **单文件分发**：简化安装，保证版本一致性
2. **编译时 DCE**：内外部构建差异化，减少包体积
3. **并行 I/O**：启动时并行执行 I/O 操作，节省约 115ms
4. **懒加载**：重型依赖延迟加载，减少初始启动时间
5. **快速路径**：常见操作（`--version`）零模块加载
