# 第 6 章：认证与凭证管理

## 6.1 认证架构概述

Claude Code 支持多种认证方式，适应不同的使用场景：

```
认证方式优先级（高 → 低）:
┌─────────────────────────────────────────────────────┐
│ 1. 文件描述符传入（FD）  ← SDK 程序化调用           │
│ 2. 环境变量 ANTHROPIC_API_KEY                       │
│ 3. OAuth Token（claude.ai 账户）                    │
│ 4. macOS Keychain 存储的 API Key                    │
│ 5. AWS Bedrock 凭证                                 │
│ 6. Google Cloud Vertex AI 凭证                      │
└─────────────────────────────────────────────────────┘
```

---

## 6.2 OAuth 2.0 + PKCE 流程

[src/services/oauth/client.ts](../../src/services/oauth/client.ts) 实现了完整的 OAuth 2.0 授权码流程，使用 PKCE（Proof Key for Code Exchange）防止授权码拦截攻击：

```
用户运行 claude login
    │
    ▼
生成 PKCE 参数：
  code_verifier = 随机 32 字节（base64url 编码）
  code_challenge = SHA256(code_verifier)（base64url 编码）
    │
    ▼
构建授权 URL：
  https://claude.ai/oauth/authorize
    ?client_id=...
    &response_type=code
    &redirect_uri=http://localhost:{port}/callback
    &code_challenge={code_challenge}
    &code_challenge_method=S256
    │
    ▼
在本地启动 HTTP 服务器监听回调
    │
    ▼
打开浏览器，用户登录 claude.ai
    │
    ▼
claude.ai 重定向到 localhost:{port}/callback?code={auth_code}
    │
    ▼
用 auth_code + code_verifier 换取 access_token：
  POST https://claude.ai/oauth/token
    {
      grant_type: 'authorization_code',
      code: auth_code,
      code_verifier: code_verifier,
      redirect_uri: ...
    }
    │
    ▼
保存 access_token + refresh_token 到 macOS Keychain
```

### 授权 URL 构建

```typescript
// src/services/oauth/client.ts
export function buildAuthUrl({
  codeChallenge,
  state,
  port,
  loginWithClaudeAi,
  inferenceOnly,
  orgUUID,
}: BuildAuthUrlParams): string {
  const authUrlBase = loginWithClaudeAi
    ? getOauthConfig().CLAUDE_AI_AUTHORIZE_URL
    : getOauthConfig().CONSOLE_AUTHORIZE_URL;

  const authUrl = new URL(authUrlBase);
  authUrl.searchParams.append('client_id', getOauthConfig().CLIENT_ID);
  authUrl.searchParams.append('response_type', 'code');
  authUrl.searchParams.append('redirect_uri', `http://localhost:${port}/callback`);
  authUrl.searchParams.append('code_challenge', codeChallenge);
  authUrl.searchParams.append('code_challenge_method', 'S256');
  // ...
  return authUrl.toString();
}
```

---

## 6.3 macOS Keychain 集成

Claude Code 使用 macOS Keychain 安全存储凭证，避免明文存储在配置文件中：

```typescript
// src/utils/secureStorage/macOsKeychainHelpers.ts
export function getMacOsKeychainStorageServiceName(): string {
  // 服务名格式：com.anthropic.claude-code
  return 'com.anthropic.claude-code';
}

// 存储凭证
await keychain.setPassword(
  serviceName,
  username,
  JSON.stringify(oauthTokens)
);

// 读取凭证
const stored = await keychain.getPassword(serviceName, username);
const tokens = JSON.parse(stored);
```

### Keychain 预取优化

[src/utils/secureStorage/keychainPrefetch.ts](../../src/utils/secureStorage/keychainPrefetch.ts) 在启动时并行预取 Keychain 数据：

```typescript
// main.tsx 启动时立即触发（与模块加载并行）
startKeychainPrefetch();

// 预取两个 Keychain 条目：
// 1. OAuth Token（新版认证）
// 2. Legacy API Key（旧版认证）

// 后续使用时直接从缓存读取，无需等待
const tokens = await ensureKeychainPrefetchCompleted();
```

---

## 6.4 Token 刷新机制

OAuth Token 有过期时间，系统自动刷新：

```typescript
// src/services/oauth/client.ts
export async function refreshOAuthToken(
  refreshToken: string
): Promise<OAuthTokens> {
  const response = await axios.post(getOauthConfig().TOKEN_URL, {
    grant_type: 'refresh_token',
    refresh_token: refreshToken,
    client_id: getOauthConfig().CLIENT_ID,
  });
  
  return {
    access_token: response.data.access_token,
    refresh_token: response.data.refresh_token,
    expires_at: Date.now() + response.data.expires_in * 1000,
  };
}

export function isOAuthTokenExpired(tokens: OAuthTokens): boolean {
  // 提前 5 分钟刷新，避免边界情况
  return Date.now() > tokens.expires_at - 5 * 60 * 1000;
}
```

---

## 6.5 多云平台认证

### AWS Bedrock

```typescript
// src/utils/aws.ts
export async function checkStsCallerIdentity(): Promise<boolean> {
  // 验证 AWS 凭证是否有效
  // 支持：环境变量、~/.aws/credentials、IAM Role
}

// 启动时预取 AWS 凭证
prefetchAwsCredentialsAndBedRockInfoIfSafe();
```

### Google Cloud Vertex AI

```typescript
// 启动时预取 GCP 凭证
prefetchGcpCredentialsIfSafe();
```

---

## 6.6 订阅类型检测

```typescript
// src/utils/auth.ts
export type SubscriptionType = 
  | 'free'
  | 'pro'
  | 'max'
  | 'team'
  | 'enterprise'
  | 'api_key_only';

export async function getSubscriptionType(): Promise<SubscriptionType> {
  // 根据 OAuth Token 的 scopes 判断订阅类型
  // 不同订阅类型有不同的速率限制和功能权限
}
```

---

## 小结

Claude Code 的认证系统设计体现了安全优先原则：
- PKCE 防止授权码拦截
- Keychain 避免明文存储
- 预取优化减少启动延迟
- 多云平台支持满足企业需求
