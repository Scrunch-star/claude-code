# 远程执行 / Teleport 设计

> 来源: V6.md 四 4.15
> 优先级: P3 (扩展系统)
> 风险: 中

```
┌───────────────────────────────────────────────────────────────────┐
│            packages/teleport (~2200行)                             │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  TeleportProvider                                           │  │
│  │  ├─ selectEnvironment(config) → Environment                 │  │
│  │  ├─ packGitContext() → Archive        (Git 打包上传)        │  │
│  │  ├─ execute(command, env) → Result    (远程执行)            │  │
│  │  └─ syncResults(remote, local)        (结果同步)            │  │
│  └─────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘
```

## 当前问题

- teleport.tsx(1234行) + teleport/(4文件, 956行) = 总计约2190行
- 散布在 utils/ 中

## 改动范围

提取为独立 package

## 依赖方向

← packages/agent; → packages/shell (远程执行)
