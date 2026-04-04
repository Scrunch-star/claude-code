# 权限系统设计

> 来源: V6.md 十
> 优先级: P2
> 风险: 中

## 权限模式 (PermissionMode)

```
PermissionMode.ts (141行)

  ┌──────────────┐  Shift+Tab 循环
  │   default    │ ←→ acceptEdits ←→ plan ←→ bypassPermissions
  │ (每步确认)   │                  ↑         (YOLO模式)
  └──────────────┘                  │
                              auto (仅内部)
                              bubble (子agent)
                              dontAsk (静默拒绝)
```

## 权限检查管线 (核心, 1486行)

`permissions.ts: hasPermissionsToUseToolInner()`

```
Tool 调用请求
     │
┌────▼────────────────────────────────────────────────────┐
│ Step 1: 强制检查 (不可被模式跳过)                         │
│                                                         │
│  1a. deny 规则匹配 → 立即拒绝                            │
│  1b. ask 规则匹配 → 需要确认                             │
│  1c. tool.checkPermissions() → 工具自检                  │
│      ├─ 文件工具 → filesystem.ts (1778行)                │
│      │   读: UNC/Windows/拒绝/工作目录/内部路径          │
│      │   写: .git/.claude/.bashrc 安全检查               │
│      └─ Bash → bashPermissions.ts (2621行)               │
│          子命令级别权限校验                                │
│  1d. 工具级拒绝                                          │
│  1e. requiresUserInteraction() → 强制交互                 │
│  1f. 内容级 ask 规则 (bypass-immune)                      │
│  1g. 安全路径检查 (.git/ .claude/ .bashrc)               │
└─────────────────────────────────────────────────────────┘
     │
┌────▼────────────────────────────────────────────────────┐
│ Step 2: 模式决策                                         │
│                                                         │
│  2a. bypassPermissions → 放行 (除非被 Step 1 拦截)       │
│  2b. alwaysAllow 规则匹配 → 放行                        │
│  2c. 无匹配 → 转为 ask                                   │
└─────────────────────────────────────────────────────────┘
     │
┌────▼────────────────────────────────────────────────────┐
│ Step 3: 模式后处理                                       │
│                                                         │
│  dontAsk   → ask 变 deny (静默拒绝)                     │
│  auto      → YOLO Classifier (1495行)                   │
│             ├─ Stage 1: 快速分类 (无 thinking)          │
│             └─ Stage 2: 深度分析 (chain-of-thought)     │
│  headless  → 先跑 PermissionRequest hooks, 再 auto-deny│
└─────────────────────────────────────────────────────────┘
     │
     ▼
结果: allow / deny / ask
```

## 规则来源 (优先级从低到高)

`permissionSetup.ts (1533行)`

```
┌────────────────┐  ┌─────────────────┐  ┌──────────────────┐
│ userSettings   │  │ projectSettings │  │ localSettings    │
│ ~/.claude/     │  │ .claude/        │  │ .claude/local    │
│ settings.json  │  │ settings.json   │  │ settings.json    │
└────────────────┘  └─────────────────┘  └──────────────────┘
┌────────────────┐  ┌─────────────────┐  ┌──────────────────┐
│ policySettings │  │ flagSettings    │  │ cliArg           │
│ (企业管理)     │  │ (GrowthBook)    │  │ --allowed-tools  │
└────────────────┘  └─────────────────┘  └──────────────────┘
┌────────────────┐  ┌─────────────────┐
│ command        │  │ session         │
│ /permissions   │  │ (临时, "始终允许"│
│                │  │  按钮产生)      │
└────────────────┘  └─────────────────┘

规则格式: "Bash(git push:*)", "Edit(.claude/**)"
解析: permissionRuleParser.ts (198行)
三种行为: allow / deny / ask
```

## 交互式审批流 (竞速模式)

`interactiveHandler.ts (536行)`

```
ask 结果到达
     │
     ▼
推送 ToolUseConfirm 到 React 确认队列
     │
     ├─────────────┬──────────────┬──────────────┐
     ▼             ▼              ▼              ▼
┌──────────┐ ┌──────────┐ ┌──────────────┐ ┌──────────────┐
│ Terminal │ │  Bridge  │ │ Channel Relay│ │ Hooks        │
│ 本地 Y/N │ │ claude.ai│ │ Telegram/IM  │ │ 外部程序审批  │
└──────────┘ └──────────┘ └──────────────┘ └──────────────┘
     │             │              │              │
     └─────────────┴──────────────┴──────────────┘
                          │
              createResolveOnce 原子竞争
              第一个响应者胜出
```

## ToolPermissionContext (核心状态对象)

存储于 AppState.toolPermissionContext

```
├─ mode: PermissionMode
├─ additionalWorkingDirectories: Map
├─ alwaysAllowRules: { [source]: string[] }
├─ alwaysDenyRules: { [source]: string[] }
├─ alwaysAskRules: { [source]: string[] }
├─ isBypassPermissionsModeAvailable: boolean
├─ isAutoModeAvailable?: boolean
├─ strippedDangerousRules?: { [source]: string[] }
├─ shouldAvoidPermissionPrompts?: boolean
└─ prePlanMode?: PermissionMode
```

## 当前问题

- 权限逻辑集中在 src/utils/permissions/ (24文件, 9416行)
- 核心文件: permissions.ts (1486行), permissionSetup.ts (1533行)
- filesystem.ts (1778行) 文件工具权限, yoloClassifier.ts (1495行) AI分类
- bashPermissions 不在 bash/ 目录, bash分类逻辑在 permissions/bashClassifier.ts
- 规则来源 7 种, 优先级隐含在加载顺序中
- UI 组件 src/components/permissions/ (79文件) 与逻辑混合
