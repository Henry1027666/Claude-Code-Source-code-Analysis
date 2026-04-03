# 第 2 章：启动流程与初始化

## 2.1 启动流程全景

当用户在终端输入 `claude` 并按下回车，一个精心设计的启动序列开始执行。整个过程被分为多个阶段，每个阶段都有性能检查点（checkpoint）记录。

```
用户输入: claude
    │
    ▼
Node.js 加载 cli.js（13MB 打包文件）
    │
    ▼
src/entrypoints/cli.tsx  ← 真正的入口
    │
    ├─ 快速路径检查（--version, --dump-system-prompt 等）
    │
    ▼
src/main.tsx  ← 主初始化逻辑
    │
    ├─ 并行启动: MDM 读取 + Keychain 预取
    ├─ 加载所有模块（~135ms）
    │
    ▼
init() 函数（src/entrypoints/init.ts）
    │
    ├─ 启用配置系统
    ├─ 应用环境变量
    ├─ 配置 TLS/mTLS/代理
    ├─ 初始化遥测
    │
    ▼
Commander.js 解析 CLI 参数
    │
    ▼
launchRepl() 或 runHeadless()
    │
    ▼
React + Ink 渲染 REPL 界面
```

---

## 2.2 入口点：cli.tsx 的快速路径设计

[src/entrypoints/cli.tsx](../../src/entrypoints/cli.tsx) 是真正的程序入口。它的设计哲学是**最小化常见路径的模块加载**：

```typescript
// cli.tsx:33-42
async function main(): Promise<void> {
  const args = process.argv.slice(2);

  // 快速路径：--version 不需要加载任何模块
  if (args.length === 1 && (args[0] === '--version' || args[0] === '-v')) {
    // MACRO.VERSION 在构建时内联
    console.log(`${MACRO.VERSION} (Claude Code)`);
    return;
  }

  // 其他路径才加载启动分析器
  const { profileCheckpoint } = await import('../utils/startupProfiler.js');
  profileCheckpoint('cli_entry');
  // ...
}
```

### 快速路径列表

| 参数 | 处理方式 | 模块加载量 |
|------|----------|-----------|
| `--version` / `-v` | 直接输出版本号退出 | 零 |
| `--dump-system-prompt` | 输出系统提示后退出 | 最小 |
| `--claude-in-chrome-mcp` | 启动 Chrome MCP 服务器 | 专用模块 |
| `--daemon-worker` | 启动守护进程工作线程 | 专用模块 |
| `remote-control` / `bridge` | 启动 Bridge 模式 | Bridge 模块 |

这种设计确保了最常见的快速操作（查看版本）几乎是即时的，而完整的 REPL 启动才需要加载全部模块。

### 编译时特性开关

```typescript
// cli.tsx:21-26
if (feature('ABLATION_BASELINE') && process.env.CLAUDE_CODE_ABLATION_BASELINE) {
  for (const k of [
    'CLAUDE_CODE_SIMPLE', 
    'CLAUDE_CODE_DISABLE_THINKING',
    'DISABLE_INTERLEAVED_THINKING',
    // ...
  ]) {
    process.env[k] ??= '1';
  }
}
```

`feature('ABLATION_BASELINE')` 是 Bun 的编译时函数——在外部发布版本中，这整个 `if` 块会被完全消除（Dead Code Elimination）。这是 Anthropic 内部 A/B 测试基础设施的一部分。

---

## 2.3 并行初始化优化

[src/main.tsx](../../src/main.tsx) 的前几行揭示了一个关键的性能优化策略：

```typescript
// main.tsx:1-20（注释说明了设计意图）
// 这些副作用必须在所有其他 import 之前运行：
// 1. profileCheckpoint 在重模块加载开始前标记入口
// 2. startMdmRawRead 启动 MDM 子进程（plutil/reg query），
//    使其与后续 ~135ms 的 import 并行运行
// 3. startKeychainPrefetch 并行启动两个 macOS keychain 读取
//    （OAuth + 旧版 API key），否则会在 applySafeConfigEnvironmentVariables()
//    内部顺序执行（每次 macOS 启动约 65ms）

import { profileCheckpoint, profileReport } from './utils/startupProfiler.js';
profileCheckpoint('main_tsx_entry');

import { startMdmRawRead } from './utils/settings/mdm/rawRead.js';
startMdmRawRead();  // 立即启动，不等待结果

import { ensureKeychainPrefetchCompleted, startKeychainPrefetch } from './utils/secureStorage/keychainPrefetch.js';
startKeychainPrefetch();  // 立即启动，不等待结果
```

### 并行化收益分析

```
串行启动（优化前）:
├─ 加载模块 (~135ms)
├─ MDM 读取 (~50ms)        ← 顺序执行
└─ Keychain 读取 (~65ms)   ← 顺序执行
总计: ~250ms

并行启动（优化后）:
├─ 加载模块 (~135ms)
│  ├─ MDM 读取 (~50ms)     ← 并行执行
│  └─ Keychain 读取 (~65ms) ← 并行执行
总计: ~135ms（节省约 115ms）
```

这是一个典型的"启动时间优化"技巧：将 I/O 密集型操作（系统调用、文件读取）与 CPU 密集型操作（模块解析）并行化。

---

## 2.4 启动性能分析器

[src/utils/startupProfiler.ts](../../src/utils/startupProfiler.ts) 实现了一个轻量级的启动性能追踪系统：

```typescript
// 采样策略：100% Anthropic 内部用户，0.5% 外部用户
const STATSIG_SAMPLE_RATE = 0.005;
const STATSIG_LOGGING_SAMPLED = 
  process.env.USER_TYPE === 'ant' || Math.random() < STATSIG_SAMPLE_RATE;

// 四个关键阶段定义
const PHASE_DEFINITIONS = {
  import_time:    ['cli_entry', 'main_tsx_imports_loaded'],  // 模块加载时间
  init_time:      ['init_function_start', 'init_function_end'],  // 初始化时间
  settings_time:  ['eagerLoadSettings_start', 'eagerLoadSettings_end'],  // 配置加载时间
  total_time:     ['cli_entry', 'main_after_run'],  // 总启动时间
};
```

### 检查点追踪

整个启动过程中散布着 `profileCheckpoint()` 调用：

```
profiler_initialized
cli_entry
main_tsx_entry
init_function_start
  init_configs_enabled
  init_safe_env_vars_applied
  init_after_graceful_shutdown
  init_after_1p_event_logging
  init_after_oauth_populate
  init_after_jetbrains_detection
  init_after_remote_settings_check
  ...
init_function_end
main_tsx_imports_loaded
eagerLoadSettings_start
eagerLoadSettings_end
main_after_run
```

开发者可以通过 `CLAUDE_CODE_PROFILE_STARTUP=1` 环境变量启用详细报告，输出每个检查点的时间和内存快照。

---

## 2.5 init() 函数：核心初始化逻辑

[src/entrypoints/init.ts](../../src/entrypoints/init.ts) 中的 `init()` 函数是系统初始化的核心，使用 `memoize` 确保只执行一次：

```typescript
export const init = memoize(async (): Promise<void> => {
  // 1. 启用配置系统
  enableConfigs();
  
  // 2. 应用安全环境变量（信任对话框之前）
  applySafeConfigEnvironmentVariables();
  
  // 3. 应用 CA 证书配置（必须在第一次 TLS 握手前）
  applyExtraCACertsFromConfig();
  
  // 4. 设置优雅关闭处理
  setupGracefulShutdown();
  
  // 5. 异步初始化（不阻塞主流程）
  void Promise.all([
    import('../services/analytics/firstPartyEventLogger.js'),
    import('../services/analytics/growthbook.js'),
  ]).then(([fp, gb]) => {
    fp.initialize1PEventLogging();
    // ...
  });
  
  // 6. 填充 OAuth 账户信息
  void populateOAuthAccountInfoIfNeeded();
  
  // 7. 初始化 JetBrains 检测
  void initJetBrainsDetection();
  
  // 8. 检测 GitHub 仓库
  void detectCurrentRepository();
  
  // 9. 初始化远程托管设置
  if (isEligibleForRemoteManagedSettings()) {
    initializeRemoteManagedSettingsLoadingPromise();
  }
  
  // 10. 配置 mTLS
  configureGlobalMTLS();
  
  // 11. 配置全局 HTTP 代理
  configureGlobalAgents();
  
  // ...更多初始化步骤
});
```

### 关键设计原则

**1. 安全优先的分阶段环境变量应用**

```typescript
// 信任对话框之前：只应用"安全"的环境变量
applySafeConfigEnvironmentVariables();

// 信任建立之后：应用完整的环境变量
applyConfigEnvironmentVariables();
```

这防止了恶意配置文件在用户确认信任之前就影响系统行为。

**2. TLS 证书的早期配置**

```typescript
// Bun 在启动时通过 BoringSSL 缓存 TLS 证书存储
// 必须在第一次 TLS 握手之前配置
applyExtraCACertsFromConfig();
```

**3. 懒加载重型模块**

```typescript
// OpenTelemetry (~400KB) 延迟加载
// gRPC (~700KB) 在 instrumentation.ts 内部进一步懒加载
void import('../services/analytics/firstPartyEventLogger.js');
```

---

## 2.6 全局状态：bootstrap/state.ts

[src/bootstrap/state.ts](../../src/bootstrap/state.ts) 定义了整个应用的全局状态结构。注释明确警告：

```typescript
// DO NOT ADD MORE STATE HERE - BE JUDICIOUS WITH GLOBAL STATE
```

### 状态结构概览

```typescript
type State = {
  // 会话标识
  originalCwd: string;       // 启动时的工作目录
  projectRoot: string;       // 项目根目录（稳定，不随 worktree 变化）
  sessionId: SessionId;      // 唯一会话 ID
  
  // 成本追踪
  totalCostUSD: number;      // 累计 API 费用
  totalAPIDuration: number;  // 累计 API 调用时间
  totalToolDuration: number; // 累计工具执行时间
  
  // 模型配置
  initialMainLoopModel: ModelSetting;
  mainLoopModelOverride: ModelSetting | undefined;
  modelStrings: ModelStrings | null;
  
  // 遥测
  meter: Meter | null;
  sessionCounter: AttributedCounter | null;
  // ...更多计数器
  
  // 权限
  bypassPermissionsMode: boolean;
  sessionBypassPermissionsMode: boolean;
  
  // 功能开关
  isInteractive: boolean;
  kairosActive: boolean;
  strictToolResultPairing: boolean;
  
  // MCP 配置
  mcpServers: McpServerConfig[];
  // ...
}
```

### 状态访问模式

状态通过导出的 getter/setter 函数访问，而非直接暴露状态对象：

```typescript
// 读取
export function getOriginalCwd(): string { return state.originalCwd; }
export function getSessionId(): SessionId { return state.sessionId; }

// 写入
export function setOriginalCwd(cwd: string): void { state.originalCwd = cwd; }
export function setCwdState(cwd: string): void { state.cwd = cwd; }
```

这种模式提供了一定的封装性，但本质上仍是全局可变状态。

---

## 2.7 配置迁移系统

[src/main.tsx](../../src/main.tsx) 中包含了大量配置迁移调用：

```typescript
// 模型名称迁移（历史遗留）
await migrateFennecToOpus();
await migrateLegacyOpusToCurrent();
await migrateOpusToOpus1m();
await migrateSonnet1mToSonnet45();
await migrateSonnet45ToSonnet46();

// 功能迁移
await migrateAutoUpdatesToSettings();
await migrateBypassPermissionsAcceptedToSettings();
await migrateEnableAllProjectMcpServersToSettings();
await migrateReplBridgeEnabledToRemoteControlAtStartup();
```

这些迁移函数处理了 Claude Code 版本升级时的配置格式变化，确保用户的历史配置能够平滑迁移到新版本。

---

## 2.8 调试保护

```typescript
// main.tsx:266-271
if ("external" !== 'ant' && isBeingDebugged()) {
  process.exit(1);
}
```

外部构建版本（`"external" !== 'ant'` 为 true）在检测到调试器时直接退出。这防止了用户通过 Node.js 调试器逆向分析内部逻辑。

`isBeingDebugged()` 检查三个来源：
1. `process.execArgv` 中的 `--inspect` / `--debug` 参数
2. `NODE_OPTIONS` 环境变量
3. Node.js `inspector` 模块的 URL（表示调试器已连接）

---

## 小结

Claude Code 的启动流程体现了几个工程最佳实践：

1. **快速路径优先**：常见操作（`--version`）零模块加载
2. **并行 I/O**：MDM 读取和 Keychain 预取与模块加载并行
3. **懒加载重型依赖**：OpenTelemetry、gRPC 等延迟到真正需要时才加载
4. **分阶段安全初始化**：先应用安全配置，信任建立后再应用完整配置
5. **性能可观测**：内置启动性能分析器，支持采样上报

下一章，我们将深入 QueryEngine——这是整个系统最核心的组件，负责驱动 AI 对话循环。
