# Swarm / 多Agent 协调设计

> 来源: V6.md 四 4.12
> 优先级: P3 (扩展系统)
> 风险: 高

```
┌───────────────────────────────────────────────────────────────────┐
│              packages/swarm (~7548行 + tasks/)                     │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  SwarmOrchestrator                                          │  │
│  │  ├─ spawnTeammate(config) → TeammateHandle                 │  │
│  │  ├─ broadcast(message)    → void                           │  │
│  │  ├─ getTeamStatus()       → TeamStatus[]                   │  │
│  │  └─ shutdown()            → void                           │  │
│  └──────────────────────────┬──────────────────────────────────┘  │
│                              │                                    │
│         ┌────────────────────┼──────────────────────┐             │
│         │                    │                      │             │
│  ┌──────▼──────────┐  ┌─────▼───────────┐  ┌──────▼──────────┐  │
│  │ InProcessBackend│  │ TmuxBackend     │  │ ITerm2Backend   │  │
│  │ (同进程, 线程)  │  │ (Tmux pane)     │  │ (iTerm2 split)  │  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
│                                                                   │
│  ┌──────────────────────┐  ┌──────────────────────────────────┐  │
│  │ TeammateMailbox      │  │ PermissionSync                   │  │
│  │ ├─ send(to, msg)     │  │ ├─ 主Agent → 子Agent 权限传递   │  │
│  │ ├─ receive()         │  │ ├─ 规则同步 / 模式继承          │  │
│  │ └─ broadcast(msg)    │  │ └─ 子Agent 结果审批             │  │
│  └──────────────────────┘  └──────────────────────────────────┘  │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  Task 类型系统 (src/tasks/, 13入口, 含巨文件)                │  │
│  │  ├─ LocalMainSessionTask  15373行 (全库最大!)               │  │
│  │  ├─ DreamTask            后台记忆合并                       │  │
│  │  ├─ InProcessTeammateTask 进程内队友                        │  │
│  │  ├─ LocalAgentTask       本地子Agent                       │  │
│  │  ├─ RemoteAgentTask      远程子Agent                       │  │
│  │  └─ MonitorMcpTask       MCP 监控                          │  │
│  └─────────────────────────────────────────────────────────────┘  │
│                                                                   │
│  ┌─────────────────────────────────────────────────────────────┐  │
│  │  Worktree 管理 (src/utils/worktree.ts, 1519行)              │  │
│  │  ├─ createWorktree() → 隔离的 git worktree                  │  │
│  │  ├─ cleanupWorktree() → 合并/丢弃                           │  │
│  │  └─ getWorktreePaths() → 路径映射                            │  │
│  └─────────────────────────────────────────────────────────────┘  │
└───────────────────────────────────────────────────────────────────┘
```

## 当前问题

- src/utils/swarm/(22文件, 7548行) + tasks/ 分散
- swarm 含 13文件(4486行) + backends/(9文件 3062行)

## 改动范围

统一到 `packages/swarm/`

## 依赖方向

← packages/agent (AgentTool); → packages/shell
