# 可扩展命令注册表：多源合并与动态加载

## 解决什么问题

AI Agent 工具需要可扩展的命令系统:
- **内置命令**: `/theme`、`/model`、`/effort` 等
- **插件命令**: 第三方插件注册的命令
- **Skills 命令**: 技能系统提供的命令
- **MCP 命令**: MCP Server 暴露的 tool 转换成的命令
- **Workflow 命令**: 用户自定义工作流

如果没有统一注册表，命令解析会散落各处，重名冲突无人处理，加载变慢。

Claude Code 实现了 **双命令平面 + 多源合并 + 按需加载** 的命令架构。

---

## 核心模式

### 模式 1: 双命令平面

Claude Code 有两个完全独立的命令系统:

```
┌──────────────────────────────────────────────────────────────┐
│  CLI 子命令平面 (外部)                                        │
│  注册: Commander program.command('mcp').action(...)           │
│  调用: shell 中 `claude mcp list`                            │
│  执行: 委托给 cli/handlers/* 模块                             │
│  生命周期: TUI 启动前执行，执行完退出                            │
├──────────────────────────────────────────────────────────────┤
│  内部 Slash 命令平面 (内部)                                    │
│  注册: getCommands() 从 commands.ts + 插件 + skills 合并       │
│  调用: REPL 内输入 `/theme`、`/model`                         │
│  执行: 直接修改 AppState、返回 JSX                             │
│  生命周期: REPL 运行期间可用                                    │
└──────────────────────────────────────────────────────────────┘
```

**CLI 子命令** 在 `main.tsx` 中注册:
```typescript
// src/main.tsx — CLI 子命令注册（只做路由）
const mcp = program.command('mcp')
  .description('Configure and manage MCP servers');

mcp.command('list').action(async () => {
  const { handleMcpList } = await import('./cli/handlers/mcp.js');
  await handleMcpList();
});

mcp.command('add').action(async (name, uri, options) => {
  const { handleMcpAdd } = await import('./cli/handlers/mcp.js');
  await handleMcpAdd(name, uri, options);
});
```

**内部 Slash 命令** 在 `commands.ts` 中定义:
```typescript
// src/commands.ts — 内置命令定义（概念简化）
function COMMANDS(): Command[] {
  return [
    themeCommand,      // /theme
    modelCommand,      // /model
    effortCommand,     // /effort
    contextCommand,    // /context
    memoryCommand,     // /memory
    // ... 50+ 命令
    // Feature-gated 命令
    ...(feature('KAIROS') ? [briefCommand] : []),
    ...(feature('BRIDGE_MODE') ? [bridgeCommand] : []),
  ];
}
```

---

### 模式 2: 多源合并去重

`getCommands()` 按优先级顺序从多个来源合并命令:

```typescript
// src/commands.ts:450-518 — 概念简化
const loadAllCommands = memoize(async (cwd: string) => {
  // 并行加载各来源
  const [skills, plugins, workflows] = await Promise.all([
    loadSkills(cwd),
    loadPlugins(cwd),
    loadWorkflows(cwd),
  ]);

  // 按优先级合并（后面的可覆盖前面的）
  return [
    ...bundledCommands,           // 1. 打包的基础命令
    ...builtinPluginCommands,     // 2. 内置插件
    ...skills,                    // 3. Skills
    ...workflows,                 // 4. Workflows
    ...pluginCommands,            // 5. 第三方插件
    ...pluginSkills,              // 6. 插件的 skills
    ...COMMANDS(),                // 7. 核心内置命令（最高优先级）
  ];
});

async function getCommands(cwd: string): Promise<Command[]> {
  const all = await loadAllCommands(cwd);
  return all
    .filter(cmd => meetsAvailabilityRequirement(cmd))  // auth/provider 门控
    .filter(cmd => isCommandEnabled(cmd))              // feature flag 门控
    .filter((cmd, i, arr) =>                           // 去重
      arr.findIndex(c => c.name === cmd.name) === i
    );
}
```

---

### 模式 3: 按 cwd 缓存加载

命令依赖工作目录（`.claude/` 配置、本地插件），所以按 `cwd` 缓存:

```typescript
const loadAllCommands = memoize(async (cwd: string) => { ... });
// 同一个 cwd 只加载一次
```

文件 watcher 检测变更时清除缓存:
```
.claude/settings.json 变更 → settingsChangeDetector.emit()
  → clearPluginCache() → loadAllCommands 下次调用时重新加载

skills/ 目录变更 → skillChangeDetector.emit()
  → 重新扫描 skills
```

---

### 模式 4: Feature Gate 编译期消除

```typescript
// src/commands.ts — feature gate 示例
const briefCommand = feature('KAIROS') || feature('KAIROS_BRIEF')
  ? require('./commands/brief.js').default
  : null;

// feature() 在构建时被替换为 true/false
// Bundler DCE (Dead Code Elimination) 移除不可达的 require
```

这意味着:
- 外部构建中 `KAIROS` 相关代码完全不存在
- 不占用 bundle 大小
- 不增加启动加载时间

---

### 模式 5: 命令处理器委托

复杂子命令把业务逻辑委托给独立模块:

```
main.tsx (命令树)          cli/handlers/ (业务逻辑)
─────────────────          ──────────────────────
program
  └── mcp
       ├── list ──────────→ handleMcpList()
       ├── add  ──────────→ handleMcpAdd()
       └── remove ────────→ handleMcpRemove()
  └── plugin
       ├── list ──────────→ handlePluginList()
       └── install ───────→ handlePluginInstall()
  └── auth
       ├── login ─────────→ handleAuthLogin()
       └── logout ────────→ handleAuthLogout()
```

好处:
- `main.tsx` 保持薄（只做路由和选项注册）
- handler 可以独立测试
- 动态 `import()` 确保只在需要时加载

---

## 内部 Slash 命令的执行流程

```
用户输入 /theme dark
    ↓
REPL 解析: 命令名 = "theme"，参数 = "dark"
    ↓
getCommands(cwd) 查找匹配命令
    ↓
命令对象: {
  name: "theme",
  description: "Switch theme",
  isAvailable: true,
  call: async (args, context) => {
    // 直接修改 AppState
    context.setAppState(s => ({ ...s, theme: args[0] }));
    // 返回 JSX 反馈
    return <Text>Theme set to {args[0]}</Text>;
  }
}
    ↓
执行 call()，渲染返回的 JSX
```

---

## 反模式

| 反模式 | 后果 | 正确做法 |
|--------|------|---------|
| 命令执行逻辑嵌入注册文件 | `main.tsx` 膨胀，难以测试 | 委托给 `cli/handlers/*` |
| 多源合并不去重 | 重名命令冲突 | `uniqBy` / `findIndex` 去重 |
| 启动时急切加载所有插件/MCP | 冷启动慢 | `memoize` 按需加载 + `feature()` DCE |
| CLI 子命令和内部 slash 命令混在一起 | 生命周期混乱 | 双命令平面分离 |
| 硬编码 feature flag 检查 | 无法在构建时消除 | `feature()` 宏 + DCE |

---

## 适用场景

- 构建支持插件/扩展的 CLI 工具
- 需要多来源命令合并（内置 + 插件 + MCP + 用户自定义）
- 需要区分"外部 CLI 命令"和"内部交互命令"
- 需要按工作目录动态加载命令配置
- 需要编译期 feature gating 控制命令可用性

---

## 关键源文件索引

| 文件 | 核心内容 |
|------|---------|
| `src/commands.ts:259-518` | 统一命令注册表、COMMANDS()、getCommands()、loadAllCommands |
| `src/main.tsx` | Commander 命令树 + preAction 钩子 + 子命令注册 |
| `src/cli/handlers/` | 委托处理器模块 (auth, mcp, plugins 等) |
| `src/commands/` | 内部 slash 命令实现 (/theme, /model 等) |
| `src/skills/` | Skills 系统 |
