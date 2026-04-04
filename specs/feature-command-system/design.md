# 命令系统设计

> 来源: V6.md 八
> 优先级: P2
> 风险: 低

## 用户输入 "/" 触发流程

```
REPL.tsx (用户按 Enter)
     │
     ▼
handlePromptSubmit.ts (610行)
     │
     ▼
processUserInput.ts (605行) ─── 识别 "/" 前缀
     │
     ▼
processSlashCommand.tsx (921行) ─── 命令分发器
     │
     │  解析命令名 + 参数
     │  findCommand() → 查找 Command 对象
     │
     ├──────────────┬──────────────────┬─────────────────┐
     ▼              ▼                  ▼                 ▼
┌─────────┐  ┌─────────────┐  ┌──────────────┐  ┌──────────┐
│  local  │  │  local-jsx  │  │    prompt    │  │ 未知命令 │
│         │  │             │  │              │  │  → 错误  │
│ load()  │  │  load()     │  │ getPrompt()  │  └──────────┘
│ →call() │  │  →call()    │  │ → 注入消息   │
│ →结果   │  │  → ReactNode│  │ → 发送查询   │
└─────────┘  └─────────────┘  └──────────────┘
     │              │                  │
     └──────────────┴──────────────────┘
                      │
                      ▼
              返回结果给 REPL
```

## 命令注册

`src/commands.ts (~470行)`

```
┌───────────────────────────────────────────────────────────────────┐
│  COMMANDS() ─── 静态注册 ~96 个命令                                │
│  (71个静态导入 + 条件feature控制 + ~25个INTERNAL_ONLY)             │
│                                                                   │
│  ┌─ src/commands/ (93个目录, 228文件) ─── 每个命令:                │
│  │  name, description, aliases, type                              │
│  │  load: () => import('./impl.js')   ← 懒加载                   │
│  │  isEnabled(), availability                                      │
│  └──────────────────────────────────────────────────────────────── │
└───────────────────────────────────────────────────────────────────┘
                      +
┌───────────────────────────────────────────────────────────────────┐
│  getCommands() ─── 动态源 (合并到最终列表)                          │
│                                                                   │
│  ├─ getSkillDirCommands()     .claude/commands/ + skills/         │
│  ├─ getBundledSkills()        内置 skill                          │
│  ├─ getPluginSkills()         插件 skill                          │
│  ├─ getPluginCommands()       插件命令                            │
│  ├─ getWorkflowCommands()     workflow 脚本命令                    │
│  └─ getDynamicSkills()        运行时动态发现                       │
└───────────────────────────────────────────────────────────────────┘
                      │
                      ▼
过滤: isEnabled → meetsAvailability → 去重
                      │
                      ▼
findCommand() / getCommand() / hasCommand()
(按 name, alias 查找, Fuse.js 模糊匹配)
```

## 自动补全

`useTypeahead (1384行)`

```
用户键入 "/"
     │
     ▼
generateCommandSuggestions() (567行)
     │
     ▼
Fuse.js 模糊搜索 ─── 精确 > alias > 前缀 > 模糊
     │
     ▼
Ghost Text 补全 / 列表选择 / Tab/Enter 确认
```

## 耦合特征

- commands.ts (~470行) 静态导入所有命令, 新增命令必须手动注册
- src/commands/ 93个目录 228文件, 每个命令独立目录
- processSlashCommand.tsx switch 分发, 新命令类型需改分发器
- SkillTool (1109行) 提供第二条路径: 模型直接调用 skill
- REMOTE_SAFE_COMMANDS / BRIDGE_SAFE_COMMANDS 硬编码安全列表
