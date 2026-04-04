# Provider 适配器层 — 执行计划

> 对应设计: `design.md`
> 优先级: P1
> 风险: 中 (LLM) / 高 (Auth)

## 当前状态快照

| 文件 | 行数 | 职责 |
|------|------|------|
| `src/services/api/claude.ts` | 3415 | 主查询函数, 含 provider if/else 分支 (~15处) |
| `src/services/api/client.ts` | 389 | `getAnthropicClient()` — 5种 SDK 构造 (Anthropic/Bedrock/Foundry/Vertex/OpenAI) |
| `src/services/api/openai/` | 882 | OpenAI 兼容层 (已验证可行, 参考模式) |
| `src/utils/model/providers.ts` | 48 | `getAPIProvider()` 类型枚举 + 环境变量选择 |
| `src/utils/auth.ts` | 2001 | 全部认证逻辑 (APIKey/OAuth/AWS/GCP/Azure/Keychain/Foundry) |
| `src/services/oauth/` | 1063 | OAuth PKCE 流程 (6文件) |

### 调用点分布

- `getAnthropicClient()`: 6 个调用点 (claude.ts×3, sideQuery, modelCapabilities, tokenEstimation, claudeAiLimits)
- `getAPIProvider()`: ~30 个调用点 (分散在 betas, system, migrations, toolSearch, effort, fastMode 等)
- `queryModelOpenAI()`: 1 个调用点 (claude.ts:1308, 动态 import)

## 目标

1. **ProviderAdapter 接口** — 统一 5 种 LLM provider 的查询入口
2. **AuthProvider 接口** — 统一 7 种认证方式的凭据获取
3. **零破坏迁移** — 现有行为不变, 仅替换内部实现

## 执行步骤

### Step 1: 定义 ProviderAdapter 接口

**文件**: `src/services/api/provider/types.ts` (新建)

```typescript
// 归一化事件格式 — 已有的 BetaRawMessageStreamEvent 子集
export interface StreamEvents {
  content_block_start: ...
  content_block_delta: ...
  content_block_stop: ...
  message_start: ...
  message_delta: ...
  message_stop: ...
  error: ...
}

// ProviderAdapter 接口
export interface ProviderAdapter {
  readonly name: string

  // 流式查询 (主路径)
  queryStreaming(params: QueryParams): AsyncGenerator<StreamEvent | AssistantMessage>

  // 非流式查询 (sideQuery 等使用)
  query(params: QueryParams): Promise<BetaMessage>

  // 可用性检查
  isAvailable(): boolean

  // 列出支持的模型
  listModels?(): Promise<ModelInfo[]>
}

export interface QueryParams {
  messages: Message[]
  systemPrompt: SystemPrompt
  tools: Tools
  signal: AbortSignal
  options: Options  // 现有 Options 类型, 逐步收敛
}
```

**改动**: 纯新增, 不影响现有代码

---

### Step 2: 实现 AnthropicAdapter (1P)

**文件**: `src/services/api/provider/anthropic.ts` (新建)

从 `claude.ts` 提取:
- 消息预处理逻辑 (normalizeMessagesForAPI → ensureToolResultPairing → stripAdvisorBlocks → stripExcessMediaItems)
- system prompt 组装 (attribution header, CLI prefix, chrome tools, deferred tools)
- `getAnthropicClient()` 调用
- 流处理循环 (content block 累积 → yield StreamEvent + AssistantMessage)
- 成本追踪

**策略**: 先用 wrapper 模式 — `AnthropicAdapter.queryStreaming()` 内部调用现有 `claude.ts` 的逻辑, 逐步将代码搬入 adapter。这避免了同时修改两处。

**依赖**: Step 1

---

### Step 3: 实现 OpenAIAdapter

**文件**: `src/services/api/provider/openai.ts` (新建)

包装现有 `src/services/api/openai/index.ts` 的 `queryModelOpenAI()`:
- 已有的 882 行代码作为 adapter 的核心实现
- 消息预处理与 Anthropic adapter 共享 (Step 2 提取出的)

**依赖**: Step 1

---

### Step 4: 实现 BedrockAdapter / VertexAdapter / FoundryAdapter

**文件**: 
- `src/services/api/provider/bedrock.ts` (新建)
- `src/services/api/provider/vertex.ts` (新建)
- `src/services/api/provider/foundry.ts` (新建)

每种 adapter 的区别点:
| | 客户端 SDK | 认证 | 区域逻辑 | 特殊 beta |
|--|-----------|------|---------|----------|
| Bedrock | `@anthropic-ai/bedrock-sdk` | AWS IAM/STS | `getAWSRegion()` | `getBedrockExtraBodyParamsBetas()` |
| Vertex | `@anthropic-ai/vertex-sdk` | GCP ADC | `getVertexRegionForModel()` | - |
| Foundry | `@anthropic-ai/foundry-sdk` | Azure AD/Managed | resource URL | - |

共同点: 都用 Anthropic SDK 变体, 事件格式相同, 无需 streamAdapter 转换。

**策略**: 先从 `client.ts:getAnthropicClient()` 的 3 个 if 分支提取, 每种 adapter 自己管理 client 创建。

**依赖**: Step 2 (共享消息预处理)

---

### Step 5: Provider 注册表

**文件**: `src/services/api/provider/registry.ts` (新建)

```typescript
const adapters: Map<string, ProviderAdapter> = new Map()

export function registerProvider(adapter: ProviderAdapter): void
export function getProvider(name?: string): ProviderAdapter  // 默认取 getAPIProvider()
export function queryWithProvider(params: QueryParams): AsyncGenerator<...>
```

替代 `claude.ts:1307` 处的 `if (getAPIProvider() === 'openai') { yield* queryModelOpenAI(...) }` — 改为 `yield* getProvider().queryStreaming(params)`。

**依赖**: Step 2-4

---

### Step 6: 切换 claude.ts 主路径

**修改**: `src/services/api/claude.ts`

1. 将 `query()` 函数的入口从直接处理改为调用 `getProvider().queryStreaming()`
2. 保留 `query()` 函数签名不变 (Options → AsyncGenerator)
3. 消息预处理逻辑移入各 adapter 或提取为共享模块

**风险控制**: 
- 保留原始 `query()` 函数作为 fallback, 通过 feature flag 切换
- `feature('PROVIDER_ADAPTER')` 控制新/旧路径
- 新路径跑通后删除旧路径

**依赖**: Step 5

---

### Step 7: 定义 AuthProvider 接口

**文件**: `src/services/api/auth/types.ts` (新建)

```typescript
export interface AuthCredentials {
  apiKey?: string
  authToken?: string        // OAuth access token
  accessToken?: string      // Bearer token (AWS/Foundry)
  // provider-specific fields
  awsAccessKey?: string
  awsSecretKey?: string
  awsSessionToken?: string
  googleAuth?: GoogleAuth
  azureADTokenProvider?: (() => Promise<string>)
}

export interface AuthProvider {
  readonly name: string
  getCredentials(): Promise<AuthCredentials>
  refresh(): Promise<void>
  isAuthenticated(): boolean
  invalidate(): void
}
```

**依赖**: 无前置

---

### Step 8: 提取 AuthProvider 实现

**文件**: `src/services/api/auth/` (新建目录)

| 实现 | 来源 | 提取内容 |
|------|------|---------|
| `apiKey.ts` | auth.ts:213-353 | `getAnthropicApiKey()`, `getApiKeyFromApiKeyHelper()` |
| `oauth.ts` | auth.ts:1193-1580 + services/oauth/ | `getClaudeAIOAuthTokens()`, `checkAndRefreshOAuthTokenIfNeeded()` |
| `awsIam.ts` | auth.ts:649-808 | `refreshAwsAuth()`, `refreshAndGetAwsCredentials()` |
| `gcpAdc.ts` | auth.ts:823-990 | `refreshGcpAuth()`, `refreshGcpCredentialsIfNeeded()` |
| `azureManaged.ts` | client.ts:191-219 | `DefaultAzureCredential` + `getBearerTokenProvider()` |
| `keychain.ts` | auth.ts:1050-1090 | `getApiKeyFromConfigOrMacOSKeychain()` |

**策略**: 每种 auth 先实现接口, 内部代理到现有函数, 避免一次性重写。

**依赖**: Step 7

---

### Step 9: AuthProvider 与 ProviderAdapter 集成

**修改**: `src/services/api/client.ts`

将 `getAnthropicClient()` 改为:
1. 根据 `getAPIProvider()` 获取对应 AuthProvider
2. AuthProvider 提供凭据
3. 用凭据构造 SDK client

原有 `getAnthropicClient()` 签名保持不变, 内部改为使用 auth/provider 注册表。

**依赖**: Step 5 + Step 8

---

### Step 10: 迁移散落的 provider 检查

**修改**: 分散在 ~30 个文件中的 `getAPIProvider()` 调用

集中到 `betas.ts`, `system.ts`, `toolSearch.ts` 等:
- 这些大多数是 provider 能力查询 (该 provider 是否支持某 beta)
- 改为 `getProvider().capabilities.supportsX` 形式
- 或在 ProviderAdapter 上暴露 `capabilities` 属性

**低优先级**, 不阻塞主路径, 可逐步迁移。

**依赖**: Step 5

---

## 文件结构 (最终)

```
src/services/api/
├── provider/
│   ├── types.ts           # ProviderAdapter 接口 + QueryParams
│   ├── registry.ts        # 注册表 + getProvider()
│   ├── anthropic.ts       # Anthropic 1P adapter
│   ├── openai.ts          # OpenAI 兼容 adapter (包装现有 openai/)
│   ├── bedrock.ts         # AWS Bedrock adapter
│   ├── vertex.ts          # GCP Vertex adapter
│   └── foundry.ts         # Azure Foundry adapter
├── auth/
│   ├── types.ts           # AuthProvider 接口 + AuthCredentials
│   ├── apiKey.ts          # API Key 认证
│   ├── oauth.ts           # OAuth PKCE 认证
│   ├── awsIam.ts          # AWS IAM/STS 认证
│   ├── gcpAdc.ts          # GCP ADC 认证
│   ├── azureManaged.ts    # Azure Managed Identity 认证
│   └── keychain.ts        # macOS Keychain 认证
├── openai/                # 保留, 被 OpenAIAdapter 引用
│   ├── streamAdapter.ts
│   ├── convertMessages.ts
│   ├── convertTools.ts
│   ├── modelMapping.ts
│   ├── client.ts
│   └── index.ts
├── claude.ts              # 逐步瘦身, 最终仅保留 query() 入口 + 预处理
└── client.ts              # getAnthropicClient() 改用 auth 注册表
```

## 迁移策略: Feature Flag

```
feature('PROVIDER_ADAPTER')
  ├─ true  → 新路径: getProvider().queryStreaming()
  └─ false → 旧路径: 现有 claude.ts 逻辑 (不变)
```

在每个切换点加入:
```typescript
if (feature('PROVIDER_ADAPTER')) {
  yield* getProvider().queryStreaming(params)
  return
}
// ... 原有逻辑保留 ...
```

## 风险缓解

| 风险 | 缓解措施 |
|------|---------|
| claude.ts 3415行提取可能破坏流处理 | 先用 wrapper 模式, 不改内部逻辑, 逐步搬入 |
| Auth 凭据缓存失效 | AuthProvider.refresh() 复用现有 memoizeWithTTLAsync |
| 5 种 SDK 构造差异大 | Anthropic 系列 (Bedrock/Vertex/Foundry) 共享基础逻辑, 仅 auth+region 不同 |
| ~30 处 getAPIProvider() 散落 | Step 10 低优先级, 不影响主路径 |
| 测试覆盖不足 | 每个 adapter 单独测试, 共享 mock fixture |

## 验收标准

- [ ] `feature('PROVIDER_ADAPTER')=1 bun run dev` 功能与旧路径一致
- [ ] 5 种 provider 各有独立 adapter, 无 if/else provider 分支
- [ ] `getAnthropicClient()` 内部使用 AuthProvider 获取凭据
- [ ] 现有 ~1623 tests 全部通过
- [ ] OpenAI 兼容层行为不变 (streamAdapter + 消息转换)
