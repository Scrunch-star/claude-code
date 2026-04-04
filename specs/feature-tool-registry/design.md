# Tool Registry 设计

> 来源: V6.md 四 4.3
> 优先级: P1
> 风险: 低

```
┌───────────────────────────────────────────┐
│              ToolRegistry                  │
│                                           │
│  register(tool)    unregister(name)       │
│  get(name)         getAll()               │
│                                           │
│  ┌─────────────────────────────────────┐  │
│  │            发现机制                  │  │
│  │  1. BuiltInTools  ─── 静态注册      │  │
│  │  2. MCPTools      ─── 动态加载 (已有)│  │
│  │  3. PluginTools   ─── npm包+配置    │  │
│  │  4. UserTools     ─── ~/.claude/    │  │
│  └─────────────────────────────────────┘  │
│                    │                       │
│                    ▼                       │
│         统一 AgentTool 接口                │
│         call() / checkPermissions()        │
│         isEnabled() / isReadOnly()         │
└───────────────────────────────────────────┘
```

## 当前问题

- tools.ts `getAllBaseTools()` 硬编码 54个 (20常驻 + 34条件, feature()控制)

## 优势

Tool 接口已很干净, MCPTool 已证明动态加载
