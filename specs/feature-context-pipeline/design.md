# Context Pipeline 设计

> 来源: V6.md 四 4.6
> 优先级: P2
> 风险: 低

```
┌──────────────────────────────────────────────────┐
│                  System Prompt 装配               │
│                                                  │
│  ┌────────────────────────────────────────────┐  │
│  │            ContextProvider[]               │  │
│  │   (按 priority 排序, 可插拔)               │  │
│  │                                            │  │
│  │   ┌─ GitStatusProvider ──── priority: 10   │  │
│  │   ├─ ClaudeMdProvider ──── priority: 20   │  │
│  │   ├─ DateProvider ──────── priority: 30   │  │
│  │   ├─ AttributionProvider ─ priority: 40   │  │
│  │   ├─ AdvisorProvider ──── priority: 50    │  │
│  │   └─ CustomProvider ────── priority: 99   │  │
│  │         (用户通过配置注册)                  │  │
│  └────────────────────────────────────────────┘  │
│                      │                           │
│                      ▼                           │
│              最终 System Prompt                   │
└──────────────────────────────────────────────────┘
```

## 当前问题

- context.ts (189行, 轻量) 提供上下文
- claude.ts buildSystemPromptBlocks() 做 prompt 缓存分块
- 无自定义 hook 点

## 改动范围

集中在 prompt 装配逻辑 (claude.ts + context.ts)
