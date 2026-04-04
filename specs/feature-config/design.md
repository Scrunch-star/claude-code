# 配置管理系统设计

> 来源: V6.md 四 4.9
> 优先级: P2 (基础设施层)
> 风险: 低-中

```
┌───────────────────────────────────────────────────────────────────┐
│                 packages/config (~9700行)                          │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  SettingsManager                                            │  │
│  │  ├─ 7 层优先级合并 (低→高, 后者覆盖前者):                   │  │
│  │  │  1. userSettings     ~/.claude/settings.json             │  │
│  │  │  2. projectSettings  .claude/settings.json               │  │
│  │  │  3. localSettings    .claude/local/settings.json         │  │
│  │  │  4. policySettings   企业管理 (远程下发)                 │  │
│  │  │  5. flagSettings     GrowthBook feature flags            │  │
│  │  │  6. cliArg           --allowed-tools 等 CLI 参数         │  │
│  │  │  7. session          临时 ("始终允许" 按钮产生)           │  │
│  │  └─ get(key) / set(key, value, source) / watch(key, cb)    │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌──────────────────────┐  ┌──────────────────────────────────┐  │
│  │ FeatureFlagProvider  │  │ SettingsSync                     │  │
│  │ ├─ BunBundle        │  │ ├─ 跨设备同步 (已部分实现)       │  │
│  │ ├─ EnvVar           │  │ ├─ 冲突检测/合并                 │  │
│  │ ├─ ConfigFile       │  │ └─ RemoteManagedSettings         │  │
│  │ ├─ Remote (GrowthBook)│  │    (企业管控配置)               │  │
│  │ └─ feature(name)→bool│  └──────────────────────────────────┘  │
│  └──────────────────────┘                                         │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  GlobalConfig (src/utils/config.ts, 1821行)                 │  │
│  │  ├─ apiKey / oauthToken / customApiKeyResponses             │  │
│  │  ├─ preferredNotifChannel / projects (per-project)          │  │
│  │  ├─ saveGlobalConfig / getGlobalConfig (文件锁+新鲜度监控)  │  │
│  │  └─ trust dialog / config backup / default factory          │  │
│  └─────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘
```

## 当前问题

- settings/(3文件 1411行) + config.ts(1821行) 混在utils
- feature() 使用181处, 分布在30+文件, 提取需谨慎

## 改动范围

提取为独立 package, 作为最底层基础设施

## 依赖方向

被 packages/agent, packages/permission 等所有模块依赖
