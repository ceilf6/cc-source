## 三. 交互

下面我想从将 claude code 视为 CLI **产品**的角度出发，看看能从其交互式系统中学习到哪些可以借鉴的思维
首先从宏观上我分为三层
1. - CLI命令与模式分流层：用户如何从外部进入系统，并被分流到正确的执行模式
2. - 终端渲染层：运行时状态如何被声明式地渲染为终端 UI
3. - 状态层：状态本身如何被组织、隔离和驱动更新
### 1. CLI命令与模式分流层

首先最外层是关注如何从**外部**进入 claude code，同时分流到正确的执行模式（至于 claude code 中例如 /theme 等等是属于 内部命令，在“应用基础设施层”展开）
我认为这一层可以学习的是其如何尽可能多实现**复用**的设计。因为 claude code CLI 支持的命令参数有很多，如何将可复用的内容固定、对差异处进行可维护的管理，想必也是各位大佬写代码很关注的一件事情
#### **通过“参数改写 + 暂存态”实现入口收敛复用**

实现方式：先在 main() 很前面解析特殊输入，把**信息**存进 _pendingConnect、_pendingAssistantChat、_pendingSSH，必要时直接改写 process.argv，然后继续走默认主流程。
	- 复用了 cc:// 的 interactive 入口：不是单独再做一套“直连 TUI 启动器”，而是把 URL 暂存后继续走默认 claude [prompt] 主线，后面统一进入 launchRepl(...)。
	- 复用了 cc:// + -p 的 headless 入口：不是重新实现一套 headless 直连解析，而是把它改写成内部 open <cc-url> 子命令，复用已有 open 处理逻辑。
	- 复用了 claude assistant [sessionId] 的交互入口：把 assistant 从 argv 里剥掉，后面仍走主交互路径，而不是再维护一套独立 UI 启动链。
	- 复用了 claude ssh <host> [dir] 的交互入口：先抽取 host/dir/flags 到 _pendingSSH，后面再在主 action 里统一决定怎么进入 REPL。
#### **通过“生命周期钩子”实现公共初始化复用**

实现方式：用 program.hook('preAction', ...) 把所有外层命令**共享的前置步骤挂成统一生命周期**，而不是散落在每个命令的 .action(...) 里。
	- 复用了 init()：默认命令和各类子命令都共用同一套基础初始化。
	- 复用了 logging sinks 初始化：避免每个子命令自己补日志接线。
	- 复用了 migration 流程：通过 runMigrations() 统一执行，而不是每个命令各自判断。
	- 复用了 remote settings / policy limits 的预热加载：命令层统一做，后面的 action 只拿结果。
	- 复用了 entrypoint 标记逻辑：initializeEntrypoint(...) 统一设置 CLAUDE_CODE_ENTRYPOINT，而不是每条分支各写一遍。
#### **通过“统一命令容器 + 委托处理器”实现命令框架复用**

实现方式：整个外层 CLI 只有一个 CommanderCommand 根对象，默认命令和子命令都挂在**同一棵命令树**下；复杂子命令再把具体逻辑**委托**给外部 handler。
也就是说命令层只做了组织和分发，复杂的执行并不会耦合在其中而是交给专门处理器
Simple 单一职责原则
	- 复用了 Commander 的解析/帮助/选项继承能力：比如 help 配置、根级 option、preAction 生命周期，不需要每个子命令重复造壳。
	- 复用了默认命令和子命令的统一注册机制：program.argument(...).action(...) 和 program.command(...).action(...) 都挂在同一个程序骨架上。
	- 复用了 handler 模块：例如 mcp 系列子命令在命令树里只负责路由，实际执行交给 cli/handlers/*，这样 main.tsx 不需要塞满业务细节。
	- 复用了命令注册片段：像 registerMcpAddCommand(...) 这种，把某一组子命令注册逻辑抽出来复用，而不是在 main.tsx 手写到底。
#### **通过“共享状态载体 + 启动契约”实现模式分流复用**

实现方式：把跨阶段要共享的数据装进统一对象，再让**多个模式分支共用同一套启动接口和上下文**。
	- 复用了 Pending* 状态对象：早期 argv 预处理阶段和后面的默认 action 阶段，不直接相互耦合，而是通过 _pendingConnect / _pendingSSH / _pendingAssistantChat 传递状态。
	- 复用了 sessionConfig：continue、resume、direct connect、ssh remote、remote 等交互分支，都尽量从同一个基础配置对象出发，只覆盖少量差异字段。
	- 复用了 resumeContext：多个恢复相关路径共享同一份恢复上下文，而不是每个恢复分支各自重新拼上下文。
	- 复用了 interactive 启动骨架：createRoot(...) -> showSetupScreens(...) -> launchRepl(...) 这条链，被多种交互模式共用。相关 helper 在 interactiveHelpers.tsx 和 replLauncher.tsx。
	- 复用了 headless 启动准备：虽然最后走的是 runHeadless(...)，但前面的 setup()、env 应用、hooks 启动、校验逻辑，和 interactive 共享了大量准备阶段。
### 2. 终端渲染层

从终端进入对话之后，问题就是：如何渲染
	- 浏览器中如果是 React 应用那么就是通过：React FiberTree -> React DOM -> Real DOM -> Browser
	- cc 的 TUI 是走的：React FiberTree -> Ink DOM -> Yoga Layout -> Screen Buffer -> Terminal
可以看到在 **render 的 reconcile** 阶段，React 仍然用同一套 Fiber/reconciliation 机制计算更新，例如像递归收集flags、diff计算等等；到了 **commit** 阶段，不再由 React DOM 把更新提交到浏览器 DOM，而是由 **Ink** 作为 **renderer** 把更新提交到它自己的终端宿主节点树，再交给 Yoga 做布局并输出到 terminal
#### React => Ink DOM

在 ink/ink.ts 中以 Ink.render(node) 为入口，调用 `react-reconciler` 的 `updateContainerSync + flushSyncWork`，触发 React 的 reconcile 中 beginWork递、completeWork归 收集flags
在 ink/reconciler.ts 中，Ink 通过 createReconciler 方法注册了一整套 host 方法，把 React 在 **commit 阶段**产生的宿主操作，逐条映射为对 **Ink DOM** 的 mutation，并同步 **Yoga 节点状态**（如 style、display、子节点结构等）
> **HostConfig** 是提供给 React 的一个**对象**，通过对象中提供的一系列**方法**，告诉 reconciler 在目标宿主环境里“怎么创建节点、怎么挂子节点、怎么更新、怎么提交”。这个宿主环境可以是 DOM、canvas、console，也可以是终端 UI
> Ink HostConfig 就相当于 React 在 终端环境 中的一个**适配器**
Ink HostConfig 对象包含的方法有如
	- `createInstance / createTextInstance`：创建 Ink DOM 节点（并按需创建/配置 Yoga 节点）
	类似于在浏览器中 display: none 的DOM节点是不配在布局树中拥有节点的，`ink-virtual-text` / `ink-link` / `ink-progress`这几类 Ink DOM 节点无需创建对应的 Yoga 节点
	- `appendChild / insertBefore / removeChild`：维护 Ink DOM 树结构，同时维护 Yoga child 列表（注意无 Yoga 节点的 child 会影响索引映射）
	- `commitUpdate / commitTextUpdate`：把 props/text 变更写入 Ink DOM；对影响布局的 style/display 等变更同步到 Yoga 节点状态（例如 `applyStyles`、`setDisplay`） -（初次挂载时）`createInstance` 内部会遍历 props 做初始化写入（Ink 内部用 helper 处理不同 prop 类别）
在 commit 的收尾时有一个钩子，会触发
1. - rootNode.onComputeLayout()
2. - rootNode.onRender?.()
这两条分别对应下面的 
	- Yoga Layout
	- 调度 frame render 使用新的 computed layout 画进 screen buffer
从 2 是在 1同步执行 之后，确保了消费数据是在数据更新之后，这也就是为什么是 Yoga Layout => Screen Buffer
> 有点像浏览器的**单线程**事件循环中 JS 执行DOM影响布局信息会**同步阻塞** HTML 解析
#### Commit 后 Yoga Layout

这一层主要聚焦于在拿到前面输入得到的元素几何信息计算“几何树”，包含每个 Yoga节点 的盒模型信息等等
我们可以主要关注其在性能优化方面的处理：类似于 React FiberTree 中有 **didReceiveUpdate **字段实现 eagerState、bailout 等性能优化策略，cc 通过 dirty 和 measure 细化了重渲染的粒度，实现了尽可能多的节点复用
	- dirty（Ink DOM）**: DOMElement.dirty**，决定 paint 阶段能否直接 blit 复用上一帧像素；`dirty=false` 允许快路径，`dirty=true` 迫使该子树重画
> blit: Block Image Transfer 块拷贝，在图形系统中表示将一块已经画好的像素区域直接复制到另一个地方，而不是重新绘制一遍。在 cc 的 TUI 场景中表示 把上一帧 screen buffer 里的一块 cell 矩形，直接复制到这一帧
	- measure（Yoga）: `ink-text/ink-raw-ansi` 这两类特定节点通过 `setMeasureFunc` 参与 Yoga 尺寸求解；当 Ink 在 `markDirty` 中对这些节点触发 `yogaNode.markDirty()` 时，会让 commit 后的 `calculateLayout` 重新进行昂贵的文本测量与换行推导
1. - 尽可能避免 dirty
 1. - `children`** 不参与 attribute 更新**
  因为 React 会给 `children` 传新引用；如果当 attribute，会导致每次都 `markDirty`。
 2. - `style`** 做值相等比较，避免每 render 触发 dirty**
  React 经常每次 render 都 `style={{...}}` new object。Ink 在 `setStyle` 里做 shallowEqual，避免无意义 `markDirty`
2. - **makeDirty** **只**对需要 re-measure **叶子**（确保是第一次遇到的）也就是我上面提到的 ink-test、ink-raw-ansi 两类节点触发 Yoga dirty
 ```typescript
 if (
   !markedYoga &&
   (current.nodeName === 'ink-text' || current.nodeName === 'ink-raw-ansi') &&
   current.yogaNode
 ) {
   current.yogaNode.markDirty()
   markedYoga = true
 }
 ```
 实现只有在 文本节点变动 才会把 脏标记 “打穿”触发昂贵的 measure
> 或许可以借鉴 [https://github.com/chenglou/pretext](https://github.com/chenglou/pretext) 思路优化 measure 过程？但是终端环境没有canvas环境并且测量对象一个是像素宽度一个是cell宽度，所以有人建议给 `prepare()` 加可插拔 `measure`，改成用 `string-width` 这类 cell 计数函数，具体可看[https://github.com/chenglou/pretext/issues/34](https://github.com/chenglou/pretext/issues/34)
#### Yoga Layout => Screen Buffer

当 Yoga 的 computed layout 已经可读后，renderer 开始消费这棵由 Ink DOM + Yoga node 组成的布局树（如果布局不存在的话会做防御，返回空frame，等到下次触发时再更新）
首先以 createRenderer 为入口，和 React 一样也用了 ping-pong **双缓冲**，目的是实现复用 Output实例（保留charCache等跨帧缓存）以降低分配与重复解析成本，然后通过布局树中根节点的 computed 尺寸确定本帧 screen 的宽高
接着进入核心递归 renderNodeToOutput ，对每个节点读其 computed rect ，判断当前节点子树能否复用上一帧对应矩形，具体用到的就是上一步拿到的 dirty 标记`若 node.dirty=false 且 rect 未变且 prevScreen 存在`那么就直接 blit 复用
核心思想和**虚拟树**一样，都是要通过先在**内存**中处理避免昂贵的开销，例如在 cc 的 TUI 中要避免的就是频繁触碰终端 I/O ，于是先将渲染结果落到**内存**中，和上一帧做比较、尽可能复用，为下一层基于 `Screen` 计算最小 patches 并写入终端做准备
#### Screen => Terminal

LogUpdate 将 prevFrame, frame.screen => 终端变更的最小patch序列
`writeDiffToTerminal` 把 patch 列表变成一次或少数几次 `stdout.write(...)`，并按终端能力决定是否包裹 **DEC 2026 同步输出**（BSU/ESU）来避免闪烁/撕裂
在交互上的体验优化措施可以归纳为：
	- **把 diff 的副作用压缩成少量 write**（减少 I/O 调用和终端重绘抖动）
	- **在支持时用同步输出保证原子性**（避免用户看到中间态）
	- **在不支持/被 tmux 破坏原子性时跳过**（避免徒增开销或错误行为）
### 3. 状态层

在知道状态如何驱动渲染更新后，现在聚焦于状态是怎么管理的
cc 并不是直接只用一个大的全局store例如 Redux 一口气收口所有状态
而是将状态按照 **频率**和**职责** 拆开了
	- 实现了 高频脏数据 与 低频共享壳层 的区分，防止高频刷新时无效的性能耗散
	- 像 React16 引入 Scheduler 一样，**缩小了更新的颗粒度**，实现了例如 messagesRef 这种需要立即可读的状态 不用等 React batching（在"REPL本地"中详细展开）
	- 并且职责不同决定了生命周期的不同，像 AppState是会话session级别，REPL本地状态是当前REPL实例级别的
拆分为下面三种状态，最后再统一交给 React+Ink 渲染
#### 全局store

全局 store 也就是 **AppState** ，承载的是会话级、共享的交互壳层状态，例如共享 UI、权限模式、MCP、插件、任务视图、footer、通知等。
在 main.tsx 中准备了 initialState ，由 launchRepl 传入到 App 后挂在了 AppStateProvider 上，这也就印证了我上面说的 AppState是顶层、会话级别的
AppStateProvider 放进 Context 的不是不断变化的 AppState，而是稳定的 store 引用，从而避免 Context value 变化导致整棵树级联重渲。
其底层实现是基于**观察者模式**：
```typescript
type Store<T> = {
  getState: () => T
  setState: (updater: (prev: T) => T) => void
  subscribe: (listener: Listener) => () => void
}
```
当 setState 更新 store 后，会通知订阅者；React 侧再通过 Context + useSyncExternalStore + selector 订阅并读取切片，只有选中值真正变化的组件才会重渲。
同时，onChangeAppState 作为 AppState 副作用的统一收口层，负责把持久化、模式同步、环境刷新等系统行为集中处理。
#### REPL本地

本地状态管理的是高频且强时序的状态，最核心的是 messages、streaming text/tool use、输入框、overlay、滚动相关。这些**更新极高频**，而且强依赖当前 REPL 生命周期，所以放在本地
正由于其高频更新的特性， cc 通过 useState + useRef 去维护，确保读取到的是**最新值**
以 messages 为例 REPL.tsx (line 1182)
```typescript
const [messages, rawSetMessages] = useState<MessageType[]>(initialMessages ?? []);
const messagesRef = useRef(messages);
const setMessages = useCallback((action: React.SetStateAction<MessageType[]>) => {
  const prev = messagesRef.current;
  const next = typeof action === 'function' ? action(messagesRef.current) : action;
  messagesRef.current = next;

  if (next.length < userInputBaselineRef.current) {
    userInputBaselineRef.current = 0;
  } else if (next.length > prev.length && userMessagePendingRef.current) {
    const delta = next.length - prev.length;
    const added =
      prev.length === 0 || next[0] === prev[0]
        ? next.slice(-delta)
        : next.slice(0, delta);

    if (added.some(isHumanTurn)) {
      userMessagePendingRef.current = false;
    } else {
      userInputBaselineRef.current = next.length;
    }
  }

  rawSetMessages(next);
}, []);
```
1. - messages 给 React 渲染用。
2. - messagesRef 给“同步立即读取最新值”的逻辑用。
> 对action做束口这部分有点像 Reducer模式 ？我记得 useReducer 和 useState 本质区别就是处理函数一个是自定义的一个是React定义的
#### 外部store

管理**跨 React/非 React** 的流程状态，像命令队列、QueryGuard、任务文件 watcher 都属于这类。它们既要被 React 订阅，又要被非 React 代码同步读写，所以独立出来最干净
具体实现是通过 模块级真相源 (如commandQueue等) + 订阅通知 + useSyncExternalStore 桥接 React
在 signal.ts 中通过 createSignal 维护 listener 集合
```typescript
export function createSignal<Args extends unknown[] = []>() {
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
listener集合会在变化时 emit，本质仍然是观察者模式，它并不负责存储状态快照，更像是负责管理订阅、广播事件的代理者，真正的外部 store 是在它上面再包一层自己的状态和 getSnapshot()
例如像命令队列commandQueue
```typescript
const commandQueue: QueuedCommand[] = []
let snapshot: readonly QueuedCommand[] = Object.freeze([])
const queueChanged = createSignal()
```
其中 commandQueue 是真实可变数据，**snapshot** 是提供给 **React** 的只读快照，queueChanged 负责在队列变化时通知订阅者。React 侧通过 useSyncExternalStore(subscribe, getSnapshot) 订阅它；而非 React 代码则可以直接调用 enqueue、dequeue、peek 等同步 API 读写这份模块级状态。
