# Server 模式设计

> 来源: V6.md 四 4.14
> 优先级: P4
> 风险: 低

```
┌───────────────────────────────────────────────────────────────────┐
│              packages/server (11文件, 大部分 stub)                  │
│               src/server/ (392行, 其中仅371行有效代码)              │
│                                                                   │
│  实际实现 (4文件, 371行):                                          │
│  ├─ directConnectManager.ts (213行) — Direct Connect 管理         │
│  ├─ createDirectConnectSession.ts (88行) — 会话创建               │
│  ├─ types.ts (57行) — 类型定义                                    │
│  └─ lockfile.ts (13行) — PID 锁                                   │
│                                                                   │
│  Stub 文件 (6个, 各3行):                                           │
│  ├─ server.ts / sessionManager.ts / connectHeadless.ts            │
│  ├─ serverBanner.ts / serverLog.ts / parseConnectUrl.ts           │
│  └─ backends/dangerousBackend.ts                                   │
└───────────────────────────────────────────────────────────────────┘
```

## 当前问题

6/11文件为空 stub, 仅 DirectConnect 有实际代码

## 建议

低优先级, 待 stub 被恢复后再考虑提取

## 依赖方向

← main.tsx (启动, DIRECT_CONNECT feature); 独立于核心
