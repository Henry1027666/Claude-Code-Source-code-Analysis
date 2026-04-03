# 第 9 章：MCP（Model Context Protocol）集成

## 9.1 MCP 协议概述

MCP（Model Context Protocol）是 Anthropic 推出的开放标准，让 Claude 能连接任意外部服务。

**没有 MCP**：Claude Code 只有 45 个内置工具
**有了 MCP**：任何服务（Slack、GitHub、数据库）只要实现 MCP 协议，Claude 就能直接调用

> 类比：MCP 就像 USB 接口——不管什么设备，只要有 USB 口就能插上用。

---

## 9.2 MCP 服务器配置

[src/services/mcp/config.ts](../../src/services/mcp/config.ts) 处理 MCP 服务器配置，支持多种来源：

```
配置优先级（高 → 低）:
┌─────────────────────────────────────────────────────┐
│ 1. 企业管理员配置（policy.json）                     │
│ 2. 项目配置（.mcp.json）                            │
│ 3. 用户配置（~/.claude/settings.json）              │
│ 4. 插件提供的配置                                   │
│ 5. Claude.ai 网页端配置                             │
└─────────────────────────────────────────────────────┘
```

### 四种连接类型

```json
{
  "mcpServers": {
    "local-tool": {
      "type": "stdio",
      "command": "python",
      "args": ["-m", "my_mcp_server"]
    },
    "remote-api": {
      "type": "http",
      "url": "https://api.example.com/mcp"
    },
    "streaming-api": {
      "type": "sse",
      "url": "https://api.example.com/sse"
    },
    "realtime": {
      "type": "ws",
      "url": "wss://api.example.com/mcp"
    }
  }
}
```

---

## 9.3 MCP 客户端实现

[src/services/mcp/client.ts](../../src/services/mcp/client.ts)（119KB，2800+ 行）是 MCP 集成的核心：

### 工具发现流程

```
程序启动
    │
    ▼
加载所有 MCP 配置
    │
    ▼
去重（手动配置优先于插件配置）
    │
    ▼
连接每个 MCP 服务器
    │
    ▼
调用 tools/list 获取工具列表
    │
    ▼
包装为 Claude 工具格式（MCPTool）
    │
    ▼
缓存到内存，供 AI 调用
```

### 工具命名规则

MCP 工具加上服务器名前缀，避免与内置工具冲突：

```
内置工具：Bash、Read、Write
MCP 工具：mcp__slack__send_message
          mcp__github__create_pr
          mcp__postgres__query
```

---

## 9.4 MCPTool 包装器

[src/tools/MCPTool/MCPTool.ts](../../src/tools/MCPTool/MCPTool.ts) 将 MCP 工具包装为标准 Claude 工具：

```typescript
export function createMCPTool(
  serverName: string,
  toolName: string,
  description: string,
  inputSchema: ToolInputJSONSchema,
  mcpClient: MCPServerConnection,
): Tool {
  return {
    name: `mcp__${serverName}__${toolName}`,
    isMcp: true,
    mcpInfo: { serverName, toolName },
    
    // MCP 工具使用 JSON Schema 而非 Zod
    inputJSONSchema: inputSchema,
    
    async call(args, context) {
      // 1. 确保连接正常（断了就重连）
      const client = await ensureConnectedClient(serverName);
      
      // 2. 发起 RPC 调用
      const result = await client.callTool({
        name: toolName,  // 去掉前缀，用原始名
        arguments: args,
      });
      
      // 3. 处理错误
      if (result.isError) {
        throw new Error(result.content[0].text);
      }
      
      return { data: result.content };
    },
    
    // 并发安全性由工具元数据决定
    isConcurrencySafe: () => !toolMeta.isDestructive,
  };
}
```

---

## 9.5 错误处理与自动重连

MCP 连接可能因各种原因断开，系统有完整的恢复机制：

| 错误类型 | 处理方式 |
|---------|---------|
| 401 认证失败 | 提示用户重新登录 |
| 404 会话过期 | 清除缓存，下次调用时重建连接 |
| 网络断开 | 触发重连，指数退避（1s → 2s → 4s…最多 30s） |
| 超时 | 最多重试 3 次 |

```typescript
// 连接缓存：复用已建立的连接
const connectionCache = new Map<string, MCPConnection>();

async function ensureConnectedClient(serverName: string) {
  const cached = connectionCache.get(serverName);
  if (cached?.isConnected()) return cached;
  
  // 重新建立连接
  const connection = await createConnection(serverName);
  connectionCache.set(serverName, connection);
  return connection;
}
```

---

## 9.6 MCP 资源（Resources）

除了工具，MCP 服务器还可以暴露**资源**——文件、文档、数据等：

```typescript
// 列出所有可用资源
// src/tools/ListMcpResourcesTool/ListMcpResourcesTool.ts
const resources = await mcpClient.listResources();
// 返回：[{ uri: 'file://docs/readme.md', name: 'README' }, ...]

// 读取某个资源
// src/tools/ReadMcpResourceTool/ReadMcpResourceTool.ts
const content = await mcpClient.readResource({ uri: 'file://docs/readme.md' });
```

---

## 9.7 MCP 认证

[src/services/mcp/auth.ts](../../src/services/mcp/auth.ts)（88KB）处理 MCP 服务器认证：

- **OAuth 2.0**：支持标准 OAuth 流程
- **自定义 Header**：API Key 等
- **XAA（Cross-App Authentication）**：Anthropic 内部认证协议

---

## 9.8 整体架构图

```
┌─────────────────────────────────────────┐
│             Claude Code                  │
│                                          │
│  内置工具（Bash、Read 等）               │
│  MCP 工具（动态发现）                    │
│  资源工具（list/read）                   │
│              ↓                           │
│       MCP 客户端管理器                   │
│  - 加载配置 / 管理连接 / 缓存工具        │
│  - 处理认证 / 错误 / 重连               │
│              ↓                           │
│  Stdio | HTTP | SSE | WebSocket          │
└──────────────┬──────────────────────────┘
               ↓
┌──────────────────────────────────────────┐
│          外部 MCP 服务器                  │
│  Slack / GitHub / 数据库 / 自定义服务    │
└──────────────────────────────────────────┘
```

---

## 小结

MCP 是 Claude Code 从"编程助手"走向"通用 AI 工作流引擎"的关键基础设施。通过标准化的协议，任何服务都可以无缝集成，无需修改 Claude Code 源码。
