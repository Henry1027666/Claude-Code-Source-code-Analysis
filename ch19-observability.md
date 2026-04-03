# 第 18 章：可观测性与遥测

## 18.1 遥测架构

Claude Code 内置了完整的可观测性基础设施：

```
遥测层次:
┌─────────────────────────────────────────────────────┐
│ OpenTelemetry（标准化遥测）                          │
│   ├─ Metrics（计数器、直方图）                       │
│   ├─ Traces（分布式追踪）                           │
│   └─ Logs（结构化日志）                             │
├─────────────────────────────────────────────────────┤
│ Statsig / GrowthBook（功能标志 + A/B 测试）          │
├─────────────────────────────────────────────────────┤
│ 自定义事件（logEvent）                              │
└─────────────────────────────────────────────────────┘
```

---

## 18.2 OpenTelemetry 集成

[src/entrypoints/init.ts](../../src/entrypoints/init.ts) 初始化 OpenTelemetry：

```typescript
// 懒加载 OpenTelemetry（~400KB），避免影响启动时间
void import('../services/analytics/firstPartyEventLogger.js').then(fp => {
  fp.initialize1PEventLogging();
});
```

### 关键指标

```typescript
// src/bootstrap/state.ts 中定义的计数器
type State = {
  // 会话计数器
  sessionCounter: AttributedCounter | null;
  
  // 代码行数计数器
  locCounter: AttributedCounter | null;
  
  // PR 计数器
  prCounter: AttributedCounter | null;
  
  // Commit 计数器
  commitCounter: AttributedCounter | null;
  
  // 成本计数器
  costCounter: AttributedCounter | null;
  
  // Token 计数器
  tokenCounter: AttributedCounter | null;
  
  // 代码编辑决策计数器
  codeEditToolDecisionCounter: AttributedCounter | null;
  
  // 活跃时间计数器
  activeTimeCounter: AttributedCounter | null;
}
```

---

## 18.3 启动性能遥测

[src/utils/startupProfiler.ts](../../src/utils/startupProfiler.ts) 采样上报启动性能：

```typescript
// 采样策略：100% Anthropic 内部用户，0.5% 外部用户
const STATSIG_SAMPLE_RATE = 0.005;

// 上报四个关键阶段的耗时
logEvent('tengu_startup_perf', {
  import_time_ms: ...,    // 模块加载时间
  init_time_ms: ...,      // 初始化时间
  settings_time_ms: ...,  // 配置加载时间
  total_time_ms: ...,     // 总启动时间
  checkpoint_count: ...,  // 检查点数量
});
```

---

## 18.4 Feature Flags（GrowthBook）

[src/services/analytics/growthbook.ts](../../src/services/analytics/growthbook.ts) 管理功能标志：

```typescript
// 检查功能标志
export function checkStatsigFeatureGate_CACHED_MAY_BE_STALE(
  gateName: string,
): boolean {
  return growthbook.isOn(gateName);
}

// 获取功能值
export function getFeatureValue_CACHED_MAY_BE_STALE<T>(
  featureName: string,
  defaultValue: T,
): T {
  return growthbook.getFeatureValue(featureName, defaultValue);
}

// GrowthBook 刷新后的回调
export function onGrowthBookRefresh(callback: () => void): void {
  growthbook.on('refresh', callback);
}
```

### 功能标志用途

| 标志 | 用途 |
|------|------|
| `tengu_tool_pear` | 工具严格模式 |
| `tengu_scratch` | 暂存区功能 |
| `tengu_1p_event_batch_config` | 事件批处理配置 |
| `COORDINATOR_MODE` | 多 Agent 协调模式 |
| `HISTORY_SNIP` | 历史记录裁剪 |

---

## 18.5 自定义事件

```typescript
// src/services/analytics/index.ts
export function logEvent(
  eventName: string,
  metadata: AnalyticsMetadata,
): void {
  // 类型系统强制要求：metadata 不能包含代码或文件路径
  // 通过 AnalyticsMetadata_I_VERIFIED_THIS_IS_NOT_CODE_OR_FILEPATHS 类型标记
  
  statsig.logEvent(eventName, null, metadata);
}
```

### 关键事件列表

| 事件 | 触发时机 |
|------|---------|
| `tengu_startup_perf` | 启动完成 |
| `tengu_model_changed` | 模型切换 |
| `tengu_managed_settings_loaded` | 托管设置加载 |
| `tengu_skill_tool_invocation` | 技能调用 |
| `tengu_permission_decision` | 权限决策 |

---

## 18.6 诊断日志

```typescript
// src/utils/diagLogs.ts
export function logForDiagnosticsNoPII(
  level: 'info' | 'warn' | 'error',
  message: string,
  data?: Record<string, unknown>,
): void {
  // 写入诊断日志文件（不包含 PII）
  // 用于调试和问题排查
  appendToLogFile({
    timestamp: Date.now(),
    level,
    message,
    data,
  });
}
```

---

## 18.7 错误追踪

```typescript
// src/utils/log.ts
export function logError(error: unknown): void {
  const err = toError(error);
  
  // 1. 写入本地日志
  logForDebugging('[ERROR]', err.message, err.stack);
  
  // 2. 上报到遥测（不包含敏感信息）
  logEvent('tengu_error', {
    error_type: err.constructor.name,
    // 注意：不上报 error.message（可能包含用户数据）
  });
}
```

---

## 小结

Claude Code 的可观测性系统是一个多层次的监控基础设施：
- **OpenTelemetry** 提供标准化的指标、追踪和日志
- **GrowthBook** 支持功能标志和 A/B 测试
- **启动性能采样** 帮助 Anthropic 持续优化启动速度
- **隐私保护** 确保遥测数据不包含用户代码或文件路径

---

# 附录：关键文件索引

## 核心文件

| 文件 | 大小 | 职责 |
|------|------|------|
| `src/main.tsx` | 主入口 | CLI 启动、参数解析、初始化 |
| `src/QueryEngine.ts` | 核心 | 会话管理、查询生命周期 |
| `src/query.ts` | 核心 | Agent 循环、工具执行 |
| `src/Tool.ts` | 接口 | 工具类型定义 |
| `src/tools.ts` | 注册 | 工具注册与管理 |
| `src/commands.ts` | 注册 | 命令注册与路由 |

## 服务层

| 文件 | 大小 | 职责 |
|------|------|------|
| `src/services/api/claude.ts` | 125KB | Claude API 客户端 |
| `src/services/mcp/client.ts` | 119KB | MCP 协议客户端 |
| `src/services/mcp/auth.ts` | 88KB | MCP 认证 |
| `src/bridge/bridgeMain.ts` | 115KB | VS Code Bridge |
| `src/bridge/replBridge.ts` | 100KB | REPL Bridge |

## UI 层

| 文件 | 大小 | 职责 |
|------|------|------|
| `src/screens/REPL.tsx` | 895KB | 主 REPL 界面 |
| `src/cli/print.ts` | 212KB | 输出格式化 |
| `src/state/AppStateStore.ts` | 状态 | 应用状态定义 |

## 工具层

| 目录 | 文件数 | 职责 |
|------|--------|------|
| `src/tools/` | 100+ | 45 个工具实现 |
| `src/commands/` | 103 | 斜杠命令 |
| `src/utils/permissions/` | 24 | 权限系统 |
| `src/utils/secureStorage/` | 多个 | 凭证管理 |
