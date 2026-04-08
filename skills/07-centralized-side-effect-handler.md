# 集中式副作用处理器：状态变更的单一收口

## 解决什么问题

在大型应用中，同一份共享状态可能被十几个不同的代码路径修改。每次修改都需要触发正确的副作用:
- 持久化到磁盘
- 通知外部服务（CCR Web UI、SDK Status Stream）
- 清除缓存（auth、credentials）
- 重新应用环境变量

如果副作用散落在各个 `setState` 调用点，**必然会遗漏路径**。

Claude Code 的源码注释记录了一个真实 bug: **8 个权限模式修改路径中有 6 个遗漏了通知 CCR**（Shift+Tab 切换、ExitPlanMode 对话框、/plan 命令、rewind、REPL bridge 等），导致 Web UI 显示的权限模式与 CLI 实际不一致。

解决方案: **在 store 层面挂一个 `onChange` 回调，ANY `setState` 调用都会自动触发**。

---

## 核心模式

### 模式 1: Wire-Once, Apply-Everywhere

`createStore` 接受一个可选的 `onChange` 回调，在每次状态变更时自动调用:

```typescript
// src/state/store.ts:10-34
export function createStore<T>(
  initialState: T,
  onChange?: OnChange<T>,       // ← 副作用收口点
): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    setState: (updater) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return
      state = next
      onChange?.({ newState: next, oldState: prev })  // ← 自动触发
      for (const listener of listeners) listener()
    },
    // ...
  }
}

// 接线: 一次即可，终身生效
const store = createStore(initialAppState, onChangeAppState);
// 之后任何 store.setState(...) 都会触发 onChangeAppState
```

---

### 模式 2: 基于 Diff 的副作用分派

`onChangeAppState` 接收 `{newState, oldState}`，对比差异决定触发哪些副作用:

```typescript
// src/state/onChangeAppState.ts:43-171
export function onChangeAppState({
  newState,
  oldState,
}: {
  newState: AppState
  oldState: AppState
}) {
  // ── 关注点 1: 权限模式同步 ──
  const prevMode = oldState.toolPermissionContext.mode
  const newMode = newState.toolPermissionContext.mode
  if (prevMode !== newMode) {
    // 外部化过滤: 内部模式(bubble, ungated auto)映射为 'default'
    const prevExternal = toExternalPermissionMode(prevMode)
    const newExternal = toExternalPermissionMode(newMode)
    if (prevExternal !== newExternal) {
      // 通知 CCR Web UI
      notifySessionMetadataChanged({ permission_mode: newExternal })
    }
    // 通知 SDK Status Stream (传原始模式，SDK 端自己过滤)
    notifyPermissionModeChanged(newMode)
  }

  // ── 关注点 2: 模型变更持久化 ──
  if (newState.mainLoopModel !== oldState.mainLoopModel) {
    if (newState.mainLoopModel === null) {
      updateSettingsForSource('userSettings', { model: undefined })
    } else {
      updateSettingsForSource('userSettings', { model: newState.mainLoopModel })
    }
    setMainLoopModelOverride(newState.mainLoopModel)
  }

  // ── 关注点 3: 展开视图持久化 ──
  if (newState.expandedView !== oldState.expandedView) {
    saveGlobalConfig(current => ({
      ...current,
      showExpandedTodos: newState.expandedView === 'tasks',
      showSpinnerTree: newState.expandedView === 'teammates',
    }))
  }

  // ── 关注点 4: verbose 持久化 ──
  if (newState.verbose !== oldState.verbose) {
    saveGlobalConfig(current => ({ ...current, verbose: newState.verbose }))
  }

  // ── 关注点 5: settings 变更 → 缓存清除 + 环境变量 ──
  if (newState.settings !== oldState.settings) {
    clearApiKeyHelperCache()
    clearAwsCredentialsCache()
    clearGcpCredentialsCache()
    if (newState.settings.env !== oldState.settings.env) {
      applyConfigEnvironmentVariables()  // 重新应用环境变量
    }
  }
}
```

---

### 模式 3: 外部化过滤

不是所有内部状态转换都应该传播到外部系统:

```typescript
// 内部权限模式     外部化后
// ──────────────  ────────
// 'default'       → 'default'
// 'bubble'        → 'default'      ← 内部临时状态
// 'ungated auto'  → 'default'      ← 内部临时状态
// 'plan'          → 'plan'
// 'auto'          → 'auto'

const prevExternal = toExternalPermissionMode(prevMode)
const newExternal = toExternalPermissionMode(newMode)
if (prevExternal !== newExternal) {
  // 只有外部可见的模式变化才通知 CCR
  notifySessionMetadataChanged({ permission_mode: newExternal })
}
```

这防止了 `default → bubble → default` 这种内部转换噪声传播到 CCR Web UI。

---

### 模式 4: 多关注点分派

单个 `onChangeAppState` 函数管理 5 个独立的关注点:

```
onChangeAppState({ newState, oldState })
    │
    ├── toolPermissionContext.mode 变化?
    │       → 通知 CCR metadata
    │       → 通知 SDK status stream
    │
    ├── mainLoopModel 变化?
    │       → 持久化到 settings
    │       → 更新 bootstrap state
    │
    ├── expandedView 变化?
    │       → 持久化到 globalConfig
    │
    ├── verbose 变化?
    │       → 持久化到 globalConfig
    │
    └── settings 变化?
            → 清除 auth 缓存
            → 重新应用环境变量
```

每个关注点通过 `if (newState.xxx !== oldState.xxx)` 独立门控，互不干扰。

---

## 为什么不散落在调用点?

来自源码的注释 (`onChangeAppState.ts:50-64`):

```
Prior to this block, mode changes were relayed to CCR by only 2 of 8+
mutation paths: a bespoke setAppState wrapper in print.ts (headless/SDK
mode only) and a manual notify in the set_permission_mode handler.
Every other path — Shift+Tab cycling, ExitPlanModePermissionRequest
dialog options, the /plan slash command, rewind, the REPL bridge's
onSetPermissionMode — mutated AppState without telling CCR, leaving
external_metadata.permission_mode stale and the web UI out of sync
with the CLI's actual mode.
```

**8 个修改路径，6 个遗漏通知**——散落式副作用的必然结果。

集中化后:
```
ANY setAppState 调用
    ↓
store.setState(updater)
    ↓
onChange({ newState, oldState })      ← 自动触发
    ↓
if (mode 变了) → 通知 CCR + SDK      ← 零遗漏
```

修改权限模式的代码**不需要知道**要通知谁。它只管修改 AppState，副作用自动发生。

---

## 反模式

| 反模式 | 后果 | 正确做法 |
|--------|------|---------|
| 在各个 `setState` 调用点添加副作用 | 必然遗漏路径 (8个路径6个遗漏) | `onChange` 集中收口 |
| 不过滤内部状态转换就通知外部 | 噪声传播 (default→bubble→default) | `toExternalPermissionMode` 过滤 |
| 副作用直接写在 React 组件中 | 组件卸载后副作用丢失 | 挂在 store 层面 |
| 不做 diff，每次都触发所有副作用 | 不必要的 I/O 和通知 | `if (new !== old)` 门控 |
| 多个 onChange 回调管理不同关注点 | 注册/注销复杂，顺序依赖 | 单个 onChange 内按关注点分块 |

---

## 适用场景

- 多个 UI 路径和非 UI 代码路径修改同一份状态
- 状态变更需要触发外部通知（API、WebSocket、日志）
- 状态变更需要持久化（文件、数据库、localStorage）
- 状态变更需要清除缓存或刷新依赖
- 需要防止遗漏副作用路径的场景

---

## 关键源文件索引

| 文件 | 核心内容 |
|------|---------|
| `src/state/onChangeAppState.ts` | 集中副作用处理器 (171行, 5个独立关注点) |
| `src/state/store.ts` | `createStore` 的 `onChange` 回调接线 |
| `src/state/AppState.tsx` | `AppStateProvider` 中 `createStore(initialState, onChangeAppState)` 接线 |
| `src/utils/permissions/PermissionMode.ts` | `toExternalPermissionMode` 外部化过滤 |
| `src/utils/sessionState.ts` | `notifySessionMetadataChanged`、`notifyPermissionModeChanged` |
