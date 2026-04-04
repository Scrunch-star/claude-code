# Hook 生命周期设计

> 来源: V6.md 四 4.7
> 优先级: P1 (核心扩展机制)
> 风险: 中

```
┌───────────────────────────────────────────────────────────────────┐
│                    HookLifecycle (packages/agent 内)              │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  Hook 点 (用户在 settings.json 配置 shell 命令)             │  │
│  │  共 27 种事件, 按类型分组:                                    │  │
│  │                                                             │  │
│  │  工具相关:                                                   │  │
│  │  ├─ PreToolUse        工具调用前 → 可阻止/修改输入          │  │
│  │  ├─ PostToolUse       工具调用后 → 可修改输出               │  │
│  │  └─ PostToolUseFailure 工具调用失败后                        │  │
│  │  会话相关:                                                   │  │
│  │  ├─ SessionStart/End   会话生命周期                         │  │
│  │  ├─ UserPromptSubmit   用户提交输入时                       │  │
│  │  ├─ Stop/StopFailure   对话停止时                           │  │
│  │  ├─ PreCompact/PostCompact  上下文压缩前后                  │  │
│  │  └─ CwdChanged         工作目录变更时                       │  │
│  │  子Agent/团队:                                              │  │
│  │  ├─ SubagentStart/Stop 子Agent生命周期                      │  │
│  │  ├─ TeammateIdle       队友空闲时                           │  │
│  │  ├─ TaskCreated/Completed 任务生命周期                      │  │
│  │  └─ WorktreeCreate/Remove Worktree管理                      │  │
│  │  权限/通知:                                                  │  │
│  │  ├─ Notification       通知触发 → 自定义渠道                │  │
│  │  ├─ PermissionRequest/Denied 权限审批                       │  │
│  │  ├─ Elicitation/Result 用户反馈请求                         │  │
│  │  └─ ConfigChange       配置变更时                           │  │
│  │  其他: Setup, InstructionsLoaded, FileChanged                │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                         │                                         │
│  ┌──────────────────────▼──────────────────────────────────────┐  │
│  │  HookExecutor (hooks.ts, 5177行 → 提取为独立模块)           │  │
│  │                                                             │  │
│  │  ├─ spawnShellCommand()  → 执行用户配置的 shell 命令        │  │
│  │  ├─ parseJsonOutput()    → 解析 hook 的 JSON 输出           │  │
│  │  ├─ timeout / abort      → 超时和中断处理                   │  │
│  │  ├─ hooksConfigSnapshot  → 配置快照 (热加载)                │  │
│  │  └─ fileChangedWatcher   → 文件变更监听触发 hook            │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  当前问题:  hooks.ts (5177行) + hooks/ (5文件 1494行) = 6713行  │
│            混杂在 utils/ 中, 27种事件类型全部硬编码               │
│  改动范围:  提取到 packages/agent/hooks/, 作为核心生命周期        │
│  依赖关系:  HookExecutor ← ToolRegistry (PreToolUse/PostToolUse) │
│            HookExecutor ← QueryEngine (Stop/SubagentStop)        │
└───────────────────────────────────────────────────────────────────┘
```

## 当前问题

- hooks.ts (5177行) + hooks/ (5文件 1494行) = 6713行
- 混杂在 utils/ 中, 27种事件类型全部硬编码

## 改动范围

提取到 `packages/agent/hooks/`, 作为核心生命周期

## 依赖关系

- HookExecutor ← ToolRegistry (PreToolUse/PostToolUse)
- HookExecutor ← QueryEngine (Stop/SubagentStop)
