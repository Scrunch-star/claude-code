# Storage Backend 设计

> 来源: V6.md 四 4.4
> 优先级: P2
> 风险: 低-中

```
┌───────────────────────────────────────────┐
│          Project (会话管理)                │
│  写队列 / 刷新 / 远程同步                  │
│                    │                       │
│         ┌──────────▼──────────┐           │
│         │   StorageBackend    │           │
│         │  read/write/append  │           │
│         │  delete/list        │           │
│         └──────────┬──────────┘           │
│                    │                       │
│    ┌───────────────┼───────────────┐      │
│    │               │               │      │
│ ┌──▼──────┐ ┌─────▼──────┐ ┌─────▼───┐  │
│ │LocalFile│ │ RemoteAPI  │ │ Memory  │  │
│ │(JSONL)  │ │(已有部分)  │ │(测试用) │  │
│ └─────────┘ └────────────┘ └─────────┘  │
└───────────────────────────────────────────┘
```

## 当前问题

- sessionStorage.ts (5106行, JSONL) + sessionRestore.ts (551行)
- sessionStoragePortable.ts (793行) + listSessionsImpl.ts (454行)
- 总计约6968行, 硬编码 JSONL 格式

## 已有基础

hydrateRemoteSession / persistToRemote 部分实现
