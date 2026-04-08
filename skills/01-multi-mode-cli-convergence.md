# Multi-Mode CLI Entry Convergence

## 解决什么问题

AI Agent CLI 工具通常需要支持多种调用模式——交互式 REPL、headless pipe（`--print`）、SSH 远程、assistant viewer、深度链接（`cc://` URL）等。如果为每种模式维护独立的启动链，会导致大量重复的初始化逻辑和难以维护的分支代码。

Claude Code 通过 **4 个核心模式** 将十余种入口收敛到统一的启动骨架中，实现了 95%+ 的基础设施复用。

---

## 核心模式

### 模式 1: Fast-Path 零导入检测

在加载完整 CLI 之前，先对特殊 flag 做零导入快路径检测，命中即返回。

```typescript
// src/entrypoints/cli.tsx:33-42
async function main(): Promise<void> {
  const args = process.argv.slice(2);

  // Fast-path for --version/-v: zero module loading needed
  if (args.length === 1 && (args[0] === '--version' || args[0] === '-v')) {
    console.log(`${MACRO.VERSION} (Claude Code)`);
    return;
  }

  // ... 更多快路径: --dump-system-prompt, --daemon-worker, remote-control,
  //     daemon, ps|logs|attach|kill, new|list|reply, environment-runner ...
  //     每个快路径只做最少的动态导入

  // 没有命中任何快路径，加载完整 CLI
  const { main: cliMain } = await import('../main.js');
  await cliMain();
}
```

**关键点**: 每个快路径分支只 `await import()` 该路径需要的模块，避免加载几百KB的完整 CLI 只为输出一行版本号。

---

### 模式 2: preAction 生命周期钩子——统一公共初始化

用 Commander 的 `program.hook('preAction', ...)` 把所有命令（默认命令 + 子命令）共享的前置步骤挂成统一生命周期，而不是散落在每个 `.action(...)` 里。

```typescript
// src/main.tsx:905-967
program.hook('preAction', async thisCommand => {
  // 1. 等待模块初始化时启动的异步子进程（~135ms 内完成）
  await Promise.all([ensureMdmSettingsLoaded(), ensureKeychainPrefetchCompleted()]);

  // 2. 核心初始化（配置验证、安全环境变量、优雅关闭）
  await init();

  // 3. 进程标题
  process.title = 'claude';

  // 4. 日志 sink 初始化（让子命令也能用 logEvent/logError）
  const { initSinks } = await import('./utils/sinks.js');
  initSinks();

  // 5. --plugin-dir 内联插件注册（顶层选项，子命令也需要）
  const pluginDir = thisCommand.getOptionValue('pluginDir');
  if (Array.isArray(pluginDir) && pluginDir.length > 0) {
    setInlinePlugins(pluginDir);
  }

  // 6. 数据迁移
  runMigrations();

  // 7. 远程配置 & 策略限制（fire-and-forget，非阻塞）
  void loadRemoteManagedSettings();
  void loadPolicyLimits();
});
```

**关键点**: `mcp`、`plugin`、`auth`、`doctor` 等子命令自动继承这些初始化步骤，零额外代码。

---

### 模式 3: Pending State + argv 改写——入口收敛

解析特殊输入后，将信息存入模块级 pending 变量，必要时改写 `process.argv`，然后继续走默认主流程。

```
入口方式          Pending 变量           后续走向
─────────────────────────────────────────────────
cc:// URL         _pendingConnect        → 默认 interactive 主线
cc:// + --print   改写 argv 为 open      → 复用 headless 逻辑
assistant [id]    _pendingAssistantChat  → 默认 interactive 主线
ssh host [dir]    _pendingSSH            → 默认 interactive 主线
--update/--upgrade 改写 argv 为 update   → 复用 update 子命令
```

实质是把"多种入口方式"降维成"默认命令 + 少量状态覆盖"，避免为每种方式维护独立启动链。

---

### 模式 4: 统一启动骨架——createRoot → showSetupScreens → launchRepl

所有交互模式最终收敛到同一条启动链，差异通过 `sessionConfig` 覆盖字段表达:

```typescript
// src/replLauncher.tsx:12-22 — 最终收敛点
export async function launchRepl(
  root: Root,
  appProps: AppWrapperProps,           // getFpsMetrics, stats, initialState
  replProps: REPLProps,                // sessionConfig, commands, tools, ...
  renderAndRun: (root: Root, element: React.ReactNode) => Promise<void>,
): Promise<void> {
  const { App } = await import('./components/App.js');
  const { REPL } = await import('./screens/REPL.js');
  await renderAndRun(root, <App {...appProps}><REPL {...replProps} /></App>);
}

// src/interactiveHelpers.tsx — 共享的 renderAndRun
export async function renderAndRun(root: Root, element: React.ReactNode): Promise<void> {
  root.render(element);
  startDeferredPrefetches();           // 首次渲染后启动后台预加载
  await root.waitUntilExit();          // 阻塞直到 REPL 卸载
  await gracefulShutdown(0);
}
```

`continue`、`resume`、SSH remote、direct connect、fresh session 等分支都调用同一个 `launchRepl()`，只是 `replProps.sessionConfig` 不同。

---

### 模式 5: 命令处理器委托

复杂子命令不在 `main.tsx` 中实现逻辑，而是委托给 `cli/handlers/*` 模块:

```typescript
// main.tsx 中的命令注册（只做路由）
mcp.command('list').action(async () => {
  const { handleMcpList } = await import('./cli/handlers/mcp.js');
  await handleMcpList();
});

// cli/handlers/mcp.ts 中的实际逻辑（可独立测试）
export async function handleMcpList(): Promise<void> {
  // 实际业务逻辑...
}
```

---

### 补充: 统一 setup() 函数

`setup.ts` 提供一个签名统一的 setup 函数，interactive 和 headless 都调用它：

```typescript
// src/setup.ts:56-66
export async function setup(
  cwd: string,
  permissionMode: PermissionMode,
  allowDangerouslySkipPermissions: boolean,
  worktreeEnabled: boolean,
  worktreeName: string | undefined,
  tmuxEnabled: boolean,
  customSessionId?: string | null,
  worktreePRNumber?: number,
  messagingSocketPath?: string,
): Promise<void> { ... }
```

内部通过 `worktreeEnabled`、`tmuxEnabled` 等布尔参数分支行为，同一函数服务不同模式。

---

## 整体流程图

```
cli.tsx (bootstrap)
    ↓ [fast-path detection]
    ├→ --version               → 零导入输出版本号 [exit]
    ├→ --daemon-worker         → 精简 worker [exit]
    ├→ remote-control          → 桥接主流程 [exit]
    ├→ daemon / ps / logs ...  → 专用处理器 [exit]
    └→ 无命中 → main.tsx
         ↓
      preAction Hook
        init() → sinks → migrations → remote settings
         ↓
      Action Handler
        pending state 消费 → sessionConfig 构建
         ↓
      ┌─────────────────┬─────────────────────┐
      │  --print/headless│    interactive       │
      │  runHeadless()   │  createRoot()        │
      │  (QueryEngine)   │  showSetupScreens()  │
      │                  │  launchRepl()         │
      └─────────────────┴─────────────────────┘
```

---

## 反模式

| 反模式 | 正确做法 |
|--------|---------|
| 为每种入口模式复制初始化逻辑 | 用 preAction 钩子统一初始化 |
| `--version` 加载完整 CLI 后判断 | 在 bootstrap 入口做零导入快路径 |
| 不同模式维护独立 UI 启动链 | 收敛到 `launchRepl()` + sessionConfig 覆盖 |
| 命令注册文件中嵌入复杂业务逻辑 | 委托给 `cli/handlers/*` 模块 |
| 同步导入所有命令处理模块 | 动态 `import()` 按需加载 |

---

## 适用场景

- 构建支持多种调用方式的 CLI 工具（交互 / pipe / 远程 / API）
- CLI 需要统一的初始化生命周期（auth、配置、日志、迁移）
- 需要将冷启动时间优化到最低（零导入快路径）
- 命令体系需要可扩展（子命令 + 内部命令 + 插件命令）

---

## 关键源文件索引

| 文件 | 核心内容 |
|------|---------|
| `src/entrypoints/cli.tsx` | Bootstrap 入口，快路径检测 |
| `src/main.tsx:905-967` | preAction 生命周期钩子 |
| `src/main.tsx:1006-2899` | Action handler，模式分流 |
| `src/setup.ts:56-66` | 统一 setup() 函数签名 |
| `src/replLauncher.tsx` | 最终收敛点 launchRepl() |
| `src/interactiveHelpers.tsx` | 共享 renderAndRun 辅助 |
| `src/cli/handlers/` | 委托处理器模块 |
