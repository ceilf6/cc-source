# 三级状态架构：按频率/职责/生命周期分层

## 解决什么问题

AI Agent 界面中共存着更新频率差异极大的状态:
- **会话配置** (permission mode, model, theme) — 几分钟甚至整个会话变一次
- **流式 token** (messages, streaming text, tool use) — 毫秒级更新
- **跨边界状态** (command queue, file watcher, task state) — 存在于 React 之外，需被 React 和非 React 代码共同访问

如果用单一全局 store，每个 token 到达都会触发所有 selector 求值、所有订阅者通知——即使它们只关心低频状态。

Claude Code 将状态拆为三个 tier，按 **更新频率 × 消费者类型 × 生命周期** 分层。

---

## 核心模式

### Tier 1: 全局 AppState（会话级 Store）

承载低频、共享的会话壳层状态: 权限模式、模型选择、UI 配置、通知、MCP 连接等。

#### 34 行的观察者模式 Store

```typescript
// src/state/store.ts — 完整实现
type Listener = () => void
type OnChange<T> = (args: { newState: T; oldState: T }) => void

export type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}

export function createStore<T>(
  initialState: T,
  onChange?: OnChange<T>,
): Store<T> {
  let state = initialState
  const listeners = new Set<Listener>()

  return {
    getState: () => state,

    setState: (updater: (prev: T) => T) => {
      const prev = state
      const next = updater(prev)
      if (Object.is(next, prev)) return  // 身份检查，避免无意义通知
      state = next
      onChange?.({ newState: next, oldState: prev })
      for (const listener of listeners) listener()
    },

    subscribe: (listener: Listener) => {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
  }
}
```

#### 稳定 Context + useSyncExternalStore + Selector

```typescript
// src/state/AppState.tsx — 概念简化

// Provider 只在挂载时创建 store，之后永远传同一个引用
function AppStateProvider({ initialState, children }) {
  const [store] = useState(() =>
    createStore(initialState, onChangeAppState)
  );
  return (
    <AppStoreContext.Provider value={store}>  {/* ← 稳定引用 */}
      {children}
    </AppStoreContext.Provider>
  );
}

// 消费者通过 selector 订阅切片
function useAppState<R>(selector: (state: AppState) => R): R {
  const store = useContext(AppStoreContext);
  const get = () => selector(store.getState());
  return useSyncExternalStore(store.subscribe, get, get);
  // 只有 selector(state) 的返回值变化时才重渲
}

// 使用
const verbose = useAppState(s => s.verbose);           // 只在 verbose 变时重渲
const model = useAppState(s => s.mainLoopModel);       // 只在 model 变时重渲
```

**关键点**: Context value 是稳定的 store 引用，不是不断变化的 state → Provider 永远不会触发子树级联重渲。

---

### Tier 2: REPL 本地状态（高频流式）

承载高频且强时序的状态: messages、streaming text/tool use、输入框、overlay、滚动。这些状态更新极高频，且强依赖当前 REPL 组件的生命周期。

#### useState + useRef 模式

```typescript
// src/screens/REPL.tsx:1182 — 概念简化
const [messages, rawSetMessages] = useState<MessageType[]>(initialMessages ?? []);
const messagesRef = useRef(messages);

const setMessages = useCallback((action: React.SetStateAction<MessageType[]>) => {
  const prev = messagesRef.current;
  const next = typeof action === 'function' ? action(messagesRef.current) : action;

  messagesRef.current = next;    // ① 立即同步更新 ref（绕过 React batching）
  rawSetMessages(next);          // ② 触发 React 重渲染
}, []);
```

**为什么需要 ref?**

React 的 `useState` 更新会被 batching 延迟。但流式回调中经常需要**立即读取最新值**:

```typescript
// 流式回调中（非 React 上下文）
function onStreamToken(token) {
  const currentMessages = messagesRef.current;  // ✅ 总是最新值
  // 而不是:
  // const currentMessages = messages;  // ❌ 闭包捕获的过时值
}
```

---

### Tier 3: 外部 Store（跨 React/非 React 边界）

承载需要被 React 组件和非 React 代码（流式处理、文件 watcher、命令队列）共同访问的状态。

#### createSignal 原语

```typescript
// src/utils/signal.ts — 完整实现
export function createSignal<Args extends unknown[] = []>(): Signal<Args> {
  const listeners = new Set<(...args: Args) => void>()
  return {
    subscribe(listener) {
      listeners.add(listener)
      return () => listeners.delete(listener)
    },
    emit(...args) {
      for (const listener of listeners) listener(...args)
    },
    clear() {
      listeners.clear()
    },
  }
}
```

**注意**: Signal 不存储状态，只管理 listener 集合和广播。它是"通知代理"，不是 store。

#### 完整的外部 Store 示例: 命令队列

```typescript
// src/utils/messageQueueManager.ts — 概念简化

// 模块级真相源
const commandQueue: QueuedCommand[] = []
let snapshot: readonly QueuedCommand[] = Object.freeze([])  // React 用的不可变快照
const queueChanged = createSignal()

// 变更时更新快照并通知
function notifySubscribers(): void {
  snapshot = Object.freeze([...commandQueue])  // 新引用 = 状态变了
  queueChanged.emit()
}

// ── 非 React 代码的同步 API ──
export function enqueue(cmd: QueuedCommand): void {
  commandQueue.push(cmd)
  notifySubscribers()
}

export function dequeue(): QueuedCommand | undefined {
  const cmd = commandQueue.shift()
  if (cmd) notifySubscribers()
  return cmd
}

// ── React 桥接 ──
export const subscribeToCommandQueue = queueChanged.subscribe
export function getCommandQueueSnapshot() { return snapshot }

// ── React Hook（1行！）──
export function useCommandQueue(): readonly QueuedCommand[] {
  return useSyncExternalStore(subscribeToCommandQueue, getCommandQueueSnapshot)
}
```

**冻结快照的妙处**: `Object.freeze` 创建新的不可变引用 → `useSyncExternalStore` 用 `Object.is` 对比引用 → 只有真正变化时才触发重渲。不需要手动做深比较。

---

## 三个 Tier 的对比

| 维度 | Tier 1: AppState | Tier 2: REPL Local | Tier 3: External Store |
|------|-----------------|-------------------|----------------------|
| **更新频率** | 低（分钟级） | 极高（毫秒级） | 中（事件驱动） |
| **生命周期** | 会话级 | REPL 实例级 | 模块级（跨组件） |
| **消费者** | React only | React + 同步回调 | React + 非 React |
| **实现** | createStore + useSyncExternalStore | useState + useRef | signal + module state + useSyncExternalStore |
| **重渲控制** | selector 切片 | 直接 setState | 冻结快照引用比较 |
| **读取方式** | `useAppState(s => s.xxx)` | `messagesRef.current` | `getCommandQueue()` / `useCommandQueue()` |

---

## 数据流向

```
用户输入
  ↓
commandQueue.enqueue()          ← Tier 3 (非 React 代码写入)
  ↓
useCommandQueue() 触发重渲      ← Tier 3 → React
  ↓
REPL 消费队列中的命令
  ↓
流式请求开始
  ↓
messagesRef.current = next      ← Tier 2 (立即同步更新)
rawSetMessages(next)             ← Tier 2 (触发 React 渲染)
  ↓
流式完成
  ↓
setAppState(s => ({...s, ...})) ← Tier 1 (持久化、通知外部)
  ↓
onChangeAppState diff            ← Tier 1 (集中副作用)
```

---

## 反模式

| 反模式 | 后果 | 正确做法 |
|--------|------|---------|
| 高频流式状态放全局 store | 每个 token 触发所有 selector 求值 | 放 REPL 本地 useState + useRef |
| `useContext` 传频繁变化的值 | Context value 变 → 整棵子树重渲 | Context 传稳定 store 引用 |
| selector 中创建新对象 | `Object.is` 永远返回 false → 每次都重渲 | selector 返回原始值或稳定引用 |
| 流式回调中从闭包读状态 | 读到过时的值 | 通过 `ref.current` 读最新值 |
| 副作用散落在各 mutation 点 | 遗漏路径导致 bug | `onChangeAppState` 集中收口 |
| 外部 store 不冻结快照 | `useSyncExternalStore` 无法检测变化 | `Object.freeze([...arr])` |

---

## 适用场景

- React 应用中有实时/流式数据（AI 流式输出、WebSocket、SSE）
- 需要在 React 和非 React 代码间共享状态
- 高频更新和低频配置共存于同一界面
- 需要精确控制组件重渲范围
- 替代 Redux 等重量级状态管理的轻量方案

---

## 关键源文件索引

| 文件 | 核心内容 |
|------|---------|
| `src/state/store.ts` | 34行 `createStore` 实现 |
| `src/state/AppState.tsx` | AppStateProvider、useAppState(selector) |
| `src/state/AppStateStore.ts` | AppState 类型定义 |
| `src/state/onChangeAppState.ts` | 集中副作用处理器 |
| `src/utils/signal.ts` | 43行 `createSignal` 原语 |
| `src/utils/messageQueueManager.ts` | 完整外部 store 示例 |
| `src/screens/REPL.tsx` | useState+useRef 高频状态模式 |
| `src/hooks/useCommandQueue.ts` | 1行 useSyncExternalStore 桥接 |
