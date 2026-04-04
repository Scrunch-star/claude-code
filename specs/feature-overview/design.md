# V6 架构重构总览

> 来源: V6.md 一、二、三、五、六、七

## 一、当前架构 (As-Is)

```
┌─────────────────────────────────────────────────────────────────┐
│                        Entry Layer                              │
│  cli.tsx → main.tsx (4680行) ─┬─→ REPL.tsx (5005行, 交互)      │
│                                ├─→ print.ts (Headless/Pipe)     │
│                                └─→ SDK (QueryEngine)            │
├─────────────────────────────────────────────────────────────────┤
│                      Core Monolith                              │
│                                                                 │
│  ┌──────────┐   ┌──────────────┐   ┌───────────────────────┐   │
│  │ query.ts │←──│ QueryEngine  │   │  AppState (199行类型)  │   │
│  │ (1732行) │   │  (1320行)    │   │  UI+MCP+Bridge+       │   │
│  │          │   │              │   │  Perm+Plugin+Agent     │   │
│  └────┬─────┘   └──────────────┘   └───────────────────────┘   │
│       │                                                         │
│  ┌────▼─────────────────────────────────────────────────────┐  │
│  │              services/api/claude.ts (3415行)              │  │
│  │  ┌─────────┐  ┌──────────┐  ┌─────────┐  ┌───────────┐  │  │
│  │  │Anthropic│  │ Bedrock  │  │ Vertex  │  │  OpenAI   │  │  │
│  │  │ (主路径)│  │(if分支)  │  │(if分支) │  │ (882行    │  │  │
│  │  │         │  │          │  │         │  │  已实现)   │  │  │
│  │  └─────────┘  └──────────┘  └─────────┘  └───────────┘  │  │
│  └──────────────────────────────────────────────────────────┘  │
│       │                                                         │
│  ┌────▼──────────┐  ┌──────────────┐  ┌──────────────────┐    │
│  │ tools.ts      │  │ services/mcp/│  │  auth.ts         │    │
│  │ (硬编码列表)  │  │  client.ts   │  │  (2001行,7种认证) │    │
│  │ 54个Tool      │  │  (3351行)    │  │  +oauth/(12文件)  │    │
│  └───────────────┘  └──────────────┘  └──────────────────┘    │
│                                                                 │
│  ┌───────────────┐  ┌───────────────┐  ┌─────────────────┐    │
│  │ sessionStorage│  │   context.ts  │  │   ink/ (104文件) │    │
│  │ (5106行,JSONL)│  │  (189行,轻量) │  │   (UI框架)      │    │
│  └───────────────┘  └───────────────┘  └─────────────────┘    │
│                                                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │  隐蔽巨文件                                               │  │
│  │  tasks/LocalMainSessionTask.ts (15373行) ← 全库最大单文件  │  │
│  │  utils/hooks.ts (5177行) + hooks/ (5文件 1494行)          │  │
│  │  components/ (596文件) ← UI组件, 低估 3.5x               │  │
│  └───────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
```

### 耦合特征

- main.tsx ~2867行内联 action handler (L1003-L3870), 51个subcommand
- REPL.tsx God Component: 54 useState, 68 useEffect, ~30 自定义Hook
- AppState 单一类型混合 7+ 个域 (UI/MCP/Permission/Bridge/Agent/Plugin/Team)
- Provider 分发靠 if/else 字符串比较 (108处 provider 相关引用)
- Tool 注册为静态列表 (getAllBaseTools), 54个工具, 无发现机制
- auth.ts (2001行) + oauth服务 (12文件 1077行) + 其他 (~779行), 总计约3857行
- hooks.ts 5177行, 27种 hook 事件类型, 混杂在 utils/ 中
- LocalMainSessionTask.ts 15373行为全库最大单体文件, 文档此前未提及

---

## 二、目标架构 (To-Be)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           Entry Layer                                   │
│                                                                         │
│   cli.tsx ──→ main.tsx (瘦身) ─┬─→ REPL (纯 UI, Hook 编排)            │
│                                 ├─→ Headless (JSON 输出)                │
│                                 ├─→ SDK (程序化接口)                    │
│                                 └─→ Deep Links (claude:// 协议)        │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  ┌──────────────────── packages/agent ──────────────────────────┐      │
│  │                  (核心引擎, 零 UI 依赖)                           │      │
│  │                                                               │      │
│  │   query()            QueryEngine          HookLifecycle       │      │
│  │   ├─ streaming       ├─ turn管理          ├─ PreToolUse       │      │
│  │   ├─ recovery        ├─ compaction        ├─ PostToolUse      │      │
│  │   ├─ attachments     ├─ SDK消息转换       ├─ Notification     │      │
│  │   └─ abort           └─ budget追踪        ├─ Stop             │      │
│  │                                          ├─ SubagentStop      │      │
│  │   CompactionService    CronScheduler      ├─ UserPromptSubmit │      │
│  │   (snip/micro/auto)    (定时任务/抖动)     ├─ SessionStart/End │      │
│  │                                          ├─ PreCompact/      │      │
│  │   LocalMainSessionTask                   │  PostCompact       │      │
│  │   (15373行 → 分解重构)                   ├─ Permission*      │      │
│  │                                          └─ ... (27种事件)    │      │
│  │   QueryDeps (依赖注入)                                        │      │
│  └──────────────────────────┬────────────────────────────────────┘      │
│                             │                                           │
├─────────────────────────────┼───────────────────────────────────────────┤
│                             ▼                                           │
│  ┌──────────────── packages/provider ────────────────────────┐         │
│  │                  (适配器层, 可扩展)                             │         │
│  │                                                             │         │
│  │  ┌─────────────────┐  ┌────────────────┐                    │         │
│  │  │ ProviderAdapter │  │  AuthProvider   │                    │         │
│  │  │ ├─queryStream() │  │  ├─getCredentials()                 │         │
│  │  │ ├─query()       │  │  ├─refresh()    │                   │         │
│  │  │ ├─isAvailable() │  │  └─invalidate() │                   │         │
│  │  │ └─listModels()  │  └────────────────┘                    │         │
│  │  └────────┬────────┘                                        │         │
│  │           │                                                  │         │
│  │  ┌────────▼────────────────────────────────────────┐        │         │
│  │  │            StreamAdapter (归一化)                │        │         │
│  │  │  将任意 SSE/WS/流 → 统一的内部事件格式          │        │         │
│  │  └─────────────────────────────────────────────────┘        │         │
│  │                                                              │         │
│  │  ┌──────────────────────────────────────────────────┐        │         │
│  │  │            ContextProvider (可插拔 prompt 管线)   │        │         │
│  │  │  GitStatus → ClaudeMd → Date → Attribution → ... │        │         │
│  │  └──────────────────────────────────────────────────┘        │         │
│  │                                                              │         │
│  │  ┌──────────────────────────────────────────────────┐        │         │
│  │  │            NetworkLayer (网络/代理)               │        │         │
│  │  │  Proxy / mTLS / CA证书 / Upstream Proxy          │        │         │
│  │  └──────────────────────────────────────────────────┘        │         │
│  └──────────────────────────────────────────────────────────────┘         │
│                             │                                             │
├─────────────────────────────┼───────────────────────────────────────────┤
│                             ▼                                             │
│  ┌──────────────── 具体实现 (Implementation) ──────────────────────┐    │
│  │                                                                 │    │
│  │  LLM Providers           Auth Implementations                   │    │
│  │  ├─ Anthropic            ├─ AnthropicOAuth                      │    │
│  │  ├─ OpenAI               ├─ APIKey (Keychain/env/config)        │    │
│  │  ├─ Gemini               ├─ AWS (Bedrock IAM)                   │    │
│  │  ├─ Mistral              ├─ GCP (Vertex ADC)                    │    │
│  │  ├─ Bedrock              └─ Azure (Managed Identity)            │    │
│  │  ├─ Vertex                                                  │    │
│  │  ├─ 通义千问             Storage Backends                       │    │
│  │  └─ 本地推理             ├─ LocalFile (JSONL)                   │    │
│  │      (llama.cpp/vLLM)    ├─ RemoteAPI                          │    │
│  │                          └─ Memory (测试)                       │    │
│  └─────────────────────────────────────────────────────────────────┘    │
│                                                                         │
├────────────────────────── UI ───────────────────────────────────────────┤
│                                                                         │
│  ┌──────────── packages/ink ───────────────────────────────────────┐   │
│  │  UI 框架 (104文件 + 扩展)                                        │   │
│  │  ├─ reconciler / hooks (useInput等) / components                │   │
│  │  ├─ Keybinding 系统    (可配置键绑定, 模式解析, 冲突解决)        │   │
│  │  ├─ Vim Emulation      (motions / operators / text objects)     │   │
│  │  ├─ Typeahead          (命令/文件建议, 模糊搜索, ghost text)    │   │
│  │  └─ InkConfig (12个注入点)                                       │   │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
├──────────────────────── 基础设施层 ─────────────────────────────────────┤
│                                                                         │
│  ┌──────── packages/agent-tools ──────────────────────────────────┐    │
│  │  Agent 工具库 (纯逻辑)                                          │    │
│  │  ├─ Tool interface + 54 个工具实现                               │    │
│  │  ├─ Sandbox 系统 (沙盒隔离执行)                                 │    │
│  │  ├─ configs, aliases, cost, deprecation, contextWindow          │    │
│  │  └─ ModelDeps (注入点)                                          │    │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌──────── packages/shell ────────────────────────────────────────┐    │
│  │  Shell 执行层 (独立抽象, 零 UI 依赖)                             │    │
│  │  ├─ ShellProvider 接口 (统一 bash/zsh/PowerShell)               │    │
│  │  ├─ Bash/Zsh 实现 (命令前缀注入/超时/环境构建)                   │    │
│  │  ├─ PowerShell 实现 (Windows 路径转换/FindGitBash)              │    │
│  │  └─ 子进程环境构建 (subprocessEnv)                               │    │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌──────── packages/config ───────────────────────────────────────┐    │
│  │  配置管理 (基础设施层, 被所有模块依赖)                            │    │
│  │  ├─ SettingsManager   (7层优先级合并)                            │    │
│  │  ├─ FeatureFlagProvider (BunBundle/EnvVar/ConfigFile/Remote)   │    │
│  │  ├─ SettingsSync       (跨设备同步)                             │    │
│  │  ├─ RemoteManagedSettings (企业管控)                             │    │
│  │  └─ GlobalConfig       (apiKey/oauthToken等, config.ts 1821行)  │    │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌──────── packages/telemetry ────────────────────────────────────┐    │
│  │  遥测/诊断 (真实实现, 非空 stub, 需谨慎处理)                      │    │
│  │  ├─ AnalyticsEventEmitter  (OTel日志导出+JSONL批处理, 806行)    │    │
│  │  ├─ GrowthBook客户端       (AB测试/FeatureFlag, 1163行)         │    │
│  │  ├─ Datadog日志            (日志上传, 321行)                    │    │
│  │  ├─ SessionTracer          (OTel兼容会话追踪)                   │    │
│  │  └─ Metadata              (事件元数据enrichment, 973行)         │    │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
├──────────────────────── 领域系统 ───────────────────────────────────────┤
│                                                                         │
│  ┌──────── packages/memory ───────────────────────────────────────┐    │
│  │  记忆系统 (独立实现, 零 UI 依赖)                                 │    │
│  │  ├─ MemoryStore (存储抽象) / MemoryRecall (相关性检索)          │    │
│  │  ├─ MemoryExtract (后台提取) / MemoryConsolidation (合并/清理)  │    │
│  │  └─ 类型: user / feedback / project / reference                 │    │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
│  ┌──────── packages/permission ───────────────────────────────────┐    │
│  │  权限系统 (独立实现, 零 UI 依赖)                                 │    │
│  │  ├─ PermissionMode (8种模式) / PermissionPipeline (检查管线)   │    │
│  │  ├─ RuleStore (allow/deny/ask 规则) / AutoClassifier (AI分类)  │    │
│  │  └─ ToolPermissionContext                                       │    │
│  └──────────────────────────────────────────────────────────────────┘   │
│                                                                         │
├──────────────────────── 扩展系统 (Phase 5) ─────────────────────────────┤
│                                                                         │
│  ┌──────────────┐  ┌──────────────────┐  ┌──────────────────────────┐  │
│  │ ToolRegistry │  │ OutputTarget     │  │ packages/swarm           │  │
│  │ ├─ 内置(静态)│  │ ├─ Terminal(Ink) │  │ (多Agent协调)            │  │
│  │ ├─ MCP(动态) │  │ ├─ JSON (SDK)    │  │ ├─ Backends              │  │
│  │ ├─ Plugin    │  │ ├─ Web (未来)    │  │ │  (进程/Tmux/iTerm2)    │  │
│  │ └─ 用户自定义│  │ └─ Silent (后台) │  │ ├─ PermissionSync        │  │
│  └──────────────┘  └──────────────────┘  │ ├─ TeammateMailbox       │  │
│                                           │ └─ Worktree管理          │  │
│  ┌──────────────────┐  ┌───────────────┐  └──────────────────────────┘  │
│  │ packages/ide     │  │ packages/server│                               │
│  │ ├─ VS Code       │  │ ├─ DirectConn │  ┌──────────────────────────┐ │
│  │ ├─ JetBrains     │  │ └─ LockFile   │  │ packages/teleport        │ │
│  │ ├─ LSP Client    │  │ (6/11文件stub │  │ ├─ 环境选择/配置         │ │
│  │ ├─ Code Indexing │  │  仅371行有效) │  │ ├─ Git 打包              │ │
│  │ └─ Claude-in-   │                      │ └─ API 集成              │ │
│  │    Chrome        │                      └──────────────────────────┘ │
│  └──────────────────┘                                                   │
│                                           ┌──────────────────────────┐ │
│  ┌──────────────────┐                     │ packages/updater        │ │
│  │ packages/cli     │                     │ ├─ NativeInstaller      │ │
│  │ ├─ Transport     │                     │ │  (.deb/.rpm/.pkg)      │ │
│  │ │  (Hybrid/SSE/  │                     │ ├─ BinaryDownload       │ │
│  │ │   WS/Worker)   │                     │ └─ AutoUpdateCheck      │ │
│  │ ├─ StructuredIO  │                     └──────────────────────────┘ │
│  │ └─ Rollback      │                                                   │
│  └──────────────────┘                                                   │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 三、单体文件分解图

```
main.tsx (4680行)                    REPL.tsx (5005行)
┌──────────────────────┐            ┌──────────────────────────┐
│ main() (安全/URL/argv)│            │    God Component         │
├──────────────────────┤            │  54 useState, 68 useEffect│
│ .action() handler     │            │  ~30 自定义 Hook         │
│ ┌──────────────────┐ │            │                          │
│ │ parseActionOptions│ │            │  ┌─ useQueryLifecycle ──┐│
│ │ (L1003-L3870,    │ │            │  │  query生命周期 (830行) ││
│ │  ~2867行内联)    │ │            │  │  权限回调, abort处理  ││
│ ├──────────────────┤ │            │  └──────────────────────┘│
│ │ mcpSetup         │ │            │  ┌─ usePromptSubmit ────┐│
│ ├──────────────────┤ │            │  │  命令解析, 队列 (350行)││
│ │ headlessSetup    │ │            │  └──────────────────────┘│
│ │ buildInitialState│ │            │  ┌─ useDialogManager ───┐│
│ ├──────────────────┤ │            │  │  通知优先级 (20路)    ││
│ │ sessionResume    │ │            │  └──────────────────────┘│
│ └──────────────────┘ │            │  ┌─ useScrollManager ───┐│
├──────────────────────┤            │  │  视口状态机 (135行)   ││
│ 51个 subcommands     │            │  └──────────────────────┘│
│ mcp/server/ssh/auth  │            │  ┌─ useSessionInit ─────┐│
│ plugin/doctor/update │            │  │  初始化, initial msg  ││
│ agents/task/autoMode │            │  └──────────────────────┘│
└──────────────────────┘            └──────────────────────────┘

query.ts (1732行)                   AppState (199行类型定义)
┌──────────────────────┐            ┌──────────────────────────┐
│ query() 异步生成器    │            │    单一嵌套类型, ~90字段  │
│ ┌──────────────────┐ │            │                          │
│ │compactionPipeline│ │            │  ┌─ UISlice ────────────┐│
│ │ (snip/micro/auto)│ │            │  │ verbose, expanded,   ││
│ ├──────────────────┤ │            │  │ footer, spinner      ││
│ │streamingOrchestrator            │  └─────────────────────┘│
│ │ (流+错误+abort)  │ │            │  ┌─ MCPSlice ───────────┐│
│ ├──────────────────┤ │            │  │ clients, tools,      ││
│ │ recovery         │ │            │  │ commands, resources  ││
│ │ (max_tokens/ptl) │ │            │  └─────────────────────┘│
│ ├──────────────────┤ │            │  ┌─ PermissionSlice ────┐│
│ │ attachments      │ │            │  │ toolPermissionContext││
│ │ (files/mem/skill)│ │            │  └─────────────────────┘│
│ └──────────────────┘ │            │  ┌─ BridgeSlice ────────┐│
└──────────────────────┘            │  │ replBridge* (~20字段)││
                                    │  └─────────────────────┘│
services/mcp/client.ts (3351行)     │  ┌─ AgentSlice ─────────┐│
┌──────────────────────┐            │  │ tasks, agents, team  ││
│ ┌──────────────────┐ │            │  └─────────────────────┘│
│ │transportManager  │ │            │  ┌─ PluginSlice ────────┐│
│ │(stdio/SSE/WS)    │ │            │  │ enabled, commands    ││
│ ├──────────────────┤ │            │  └─────────────────────┘│
│ │ toolDiscovery    │ │            └──────────────────────────┘
│ │(MCPTool实例化)   │ │
│ ├──────────────────┤ │            → Domain Slicing:
│ │ authManager      │ │            type AppState = UISlice
│ │(OAuth+重连)      │ │              & MCPSlice & PermissionSlice
│ └──────────────────┘ │              & BridgeSlice & AgentSlice
└──────────────────────┘              & PluginSlice & TeamSlice

LocalMainSessionTask.ts (15373行)    ← 全库最大单体文件
┌──────────────────────┐            (此前文档未提及)
│ 单文件承担本地主会话   │
│ 全部生命周期管理      │
│ 包含: turn管理/消息   │
│ 处理/工具调用/权限    │
│ 恢复/compaction等    │
│ 分解优先级: P1        │
└──────────────────────┘
```

---

## 五、Packages 目录结构

```
packages/
├── @ant/                           (已有)
│   ├── computer-use-mcp/
│   ├── computer-use-input/
│   ├── computer-use-swift/
│   └── claude-for-chrome-mcp/
├── audio-capture-napi/             (已有)
├── color-diff-napi/                (已有)
├── image-processor-napi/           (已有)
│
├── ink/               ← Phase 1   UI 框架 + Keybinding + Vim + Typeahead
├── agent-tools/        ← Phase 2   Agent 工具库 + Sandbox 系统
├── memory/             ← Phase 2   记忆系统, 独立实现
├── permission/         ← Phase 2   权限系统, 独立实现
├── config/             ← Phase 2   配置管理 + FeatureFlag + GlobalConfig
├── telemetry/          ← Phase 2   遥测/诊断 (真实实现, 非空stub)
├── agent/              ← Phase 3   核心引擎 + Hook 生命周期 + Compaction + Cron
├── provider/           ← Phase 4   ProviderAdapter + AuthProvider + NetworkLayer
├── shell/              ← Phase 4   Shell 执行层 (Bash/Zsh/PowerShell)
├── swarm/              ← Phase 5   多Agent协调 + Worktree
├── ide/                ← Phase 5   IDE/LSP/CodeIndex + Claude-in-Chrome
├── server/             ← Phase 5   服务器模式 (大部分stub, 低优先级)
├── teleport/           ← Phase 5   远程执行环境
├── updater/            ← Phase 5   自动更新 + 原生安装器
└── cli/                ← Phase 5   CLI 传输层 + Handler 分发
```

---

## 六、实施路线图

```
Phase 0: 内部分解 (与 Phase 1-3 并行, 低风险)
  ├── main.tsx  → 6 个模块
  ├── REPL.tsx  → 5 个 Hook
  ├── query.ts  → 4 个子模块
  ├── services/mcp/client.ts → 3 个模块
  ├── LocalMainSessionTask.ts → 分解 (15373行, 全库最大)
  └── AppState → Domain Slicing

Phase 1: packages/ink/                    风险: 低
  ├── Ink 框架 (reconciler/hooks/components)
  ├── Keybinding 系统 (16文件 → 纳入 ink/)
  ├── Vim 模拟 (5文件 → 纳入 ink/)
  └── Typeahead/Suggestion (→ 纳入 ink/)

Phase 2: 独立系统提取                          风险: 低-中
  ├── packages/agent-tools/    Agent 工具库 + Sandbox
  ├── packages/memory/         记忆系统
  ├── packages/permission/     权限系统
  ├── packages/config/         配置管理 + FeatureFlag + GlobalConfig
  └── packages/telemetry/      遥测/诊断 (真实实现, 谨慎提取)

Phase 3: packages/agent/                  风险: 中-高
  ├── query() + QueryEngine 核心循环
  ├── Hook 生命周期 (hooks.ts 5177行 + hooks/ 5文件 → 提取, 27种事件)
  ├── Compaction 服务 (services/compact/ 29文件 → 统一)
  ├── Cron/Scheduler (utils/cron* → 提取)
  └── FileHistory (utils/fileHistory → 提取)

Phase 4: packages/provider/ + packages/shell/  风险: 中
  ├── LLM Provider 适配器 (核心价值最高)
  ├── Auth Provider 适配器
  ├── Context Pipeline
  ├── NetworkLayer (proxy/mTLS/CA证书)
  └── Shell 执行层 (bash/powershell → 提取)

Phase 5: 扩展系统                           风险: 中-高
  ├── Tool Registry / Plugin
  ├── Storage Backend / Command System
  ├── Output Target / Feature Flag Provider
  ├── packages/swarm/     多Agent协调 + Worktree
  ├── packages/ide/       IDE/LSP + CodeIndex + Chrome
  ├── packages/server/    服务器 (大部分stub, 可延后)
  ├── packages/teleport/  远程执行
  ├── packages/updater/   自动更新 + 安装器
  └── packages/cli/       CLI 传输层
```

---

## 七、优先级矩阵

| 项目 | 用户价值 | 风险 | 优先级 |
|------|---------|------|--------|
| **LLM Provider 适配器** | 高 (多模型, 已有OpenAI参考实现) | 中 | **P1** |
| **Auth Provider 适配器** | 高 (安全, 7种认证约3857行) | 高 | **P1** |
| **Tool Registry** | 高 (生态, 54个工具) | 低 | **P1** |
| **Hook 生命周期** | 高 (核心扩展点, 27种事件) | 中 | **P1** |
| **LocalMainSessionTask分解** | 高 (15373行巨文件) | 中 | **P1** |
| **Shell 执行层** | 高 (BashTool底层, bash/12310行) | 中 | **P2** |
| **配置管理** | 高 (基础设施, config.ts 1821行) | 低-中 | **P2** |
| **记忆系统** | 高 (跨会话记忆, 24文件 6330行) | 低-中 | **P2** |
| **权限系统** | 高 (安全/可扩展, 24文件 9416行) | 中 | **P2** |
| **Compaction 服务** | 中 (上下文管理, 29文件 4267行) | 低 | **P2** |
| **命令系统** | 中 (可扩展命令, 93目录 96命令) | 低 | P2 |
| Context Pipeline | 中 (自定义prompt) | 低 | P2 |
| Storage Backend | 中 (云同步/测试, 5文件约6968行) | 低-中 | P2 |
| **遥测/诊断** | 中 (可观测性, 真实实现非stub) | 中 | P2 |
| main.tsx 分解 | 中 (可维护性, 4680行) | 低 | P2 |
| REPL.tsx 分解 | 中 (可维护性, 5005行) | 中 | P2 |
| query.ts 分解 | 中 (可维护性, 1732行) | 低 | P3 |
| AppState Slicing | 低 (199行类型) | 低 | P3 |
| mcp/client.ts 分解 | 低 (3351行, services/mcp/) | 低 | P3 |
| Output Target | 中 (596个组件, 范围大) | 中-高 | P3 |
| Feature Flag Provider | 低 (181处使用, 30+文件) | 中 | P3 |
| **Swarm/多Agent** | 高 (并行能力, 22文件 7548行) | 高 | **P3** |
| **IDE/LSP 集成** | 中 (编辑器体验) | 中 | P3 |
| **Server 模式** | 低 (6/11文件为stub, 仅371行有效) | 低 | P4 |
| **Teleport 远程执行** | 中 (远程开发, 约2190行) | 中 | P3 |
| **自动更新/安装器** | 低 (运维, 约3579行) | 低 | P3 |
| **CLI 传输层** | 低 (内部架构, 127文件) | 低 | P3 |
