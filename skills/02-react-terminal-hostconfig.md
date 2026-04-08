# React-to-Terminal HostConfig Adapter

## 解决什么问题

React 的 reconciler 通过抽象的 **HostConfig** 接口与宿主环境交互。浏览器中由 React DOM 提供这套接口；终端环境中需要自行实现，将 React 的 commit 阶段操作映射为对**终端节点树**的 mutation，并同步**布局引擎**(Yoga) 的节点状态。

Claude Code 通过 Ink 实现了完整的终端 HostConfig，渲染管线为:

```
React FiberTree → Ink DOM → Yoga Layout → Screen Buffer → Terminal
```

---

## 核心模式

### 模式 1: HostConfig 方法映射

`reconciler.ts` 通过 `createReconciler({...})` 注册了完整的 host 方法集，将 React commit 阶段的宿主操作映射为 Ink DOM mutation:

```typescript
// src/ink/reconciler.ts:224-506
const reconciler = createReconciler({
  // 创建节点
  createInstance(originalType, newProps, _root, hostContext): DOMElement {
    // ink-text 嵌套在 text 内部时自动提升为 ink-virtual-text（无需 Yoga 节点）
    const type = originalType === 'ink-text' && hostContext.isInsideText
      ? 'ink-virtual-text'
      : originalType;
    const node = createNode(type);
    for (const [key, value] of Object.entries(newProps)) {
      applyProp(node, key, value);
    }
    return node;
  },

  createTextInstance(text, _root, hostContext): TextNode {
    if (!hostContext.isInsideText) {
      throw new Error(`Text string "${text}" must be rendered inside <Text> component`);
    }
    return createTextNode(text);
  },

  // 树操作
  appendChild: appendChildNode,
  insertBefore: insertBeforeNode,
  removeChild(node, removeNode) {
    removeChildNode(node, removeNode);
    cleanupYogaNode(removeNode);  // 释放 WASM 内存
  },

  // Props 变更
  commitUpdate(node, _type, oldProps, newProps): void {
    const props = diff(oldProps, newProps);
    if (props) {
      for (const [key, value] of Object.entries(props)) {
        if (key === 'style') setStyle(node, value);
        else if (key === 'textStyles') setTextStyles(node, value);
        else if (EVENT_HANDLER_PROPS.has(key)) setEventHandler(node, key, value);
        else setAttribute(node, key, value);
      }
    }
    // 同步 Yoga 节点样式
    const style = diff(oldProps['style'], newProps['style']);
    if (style && node.yogaNode) {
      applyStyles(node.yogaNode, style, newProps['style']);
    }
  },

  commitTextUpdate(node, _oldText, newText): void {
    setTextNodeValue(node, newText);
  },

  // Commit 阶段收尾——触发布局计算和帧渲染
  resetAfterCommit(rootNode) {
    rootNode.onComputeLayout?.();  // → yogaNode.calculateLayout()
    rootNode.onRender?.();          // → 调度帧渲染（throttled）
  },
  // ...
});
```

---

### 模式 2: 选择性 Yoga 节点创建

不是每个 Ink DOM 节点都需要 Yoga 布局节点。类似于浏览器中 `display: none` 的元素不参与布局树:

```typescript
// src/ink/dom.ts:110-132
export const createNode = (nodeName: ElementNames): DOMElement => {
  const needsYogaNode =
    nodeName !== 'ink-virtual-text' &&  // 嵌套 text，不参与布局
    nodeName !== 'ink-link' &&           // 超链接包装，不参与布局
    nodeName !== 'ink-progress'          // 进度条特殊处理

  const node: DOMElement = {
    nodeName,
    style: {},
    attributes: {},
    childNodes: [],
    parentNode: undefined,
    yogaNode: needsYogaNode ? createLayoutNode() : undefined,
    dirty: false,
  };

  // 文本节点绑定测量函数
  if (nodeName === 'ink-text') {
    node.yogaNode?.setMeasureFunc(measureTextNode.bind(null, node));
  } else if (nodeName === 'ink-raw-ansi') {
    node.yogaNode?.setMeasureFunc(measureRawAnsiNode.bind(null, node));
  }
  return node;
};
```

**关键影响**: `insertBefore` 需要跳过无 Yoga 节点的 child 重新计算索引:

```typescript
// src/ink/dom.ts:155-201
export const insertBeforeNode = (node, newChildNode, beforeChildNode): void => {
  const index = node.childNodes.indexOf(beforeChildNode);
  if (index >= 0) {
    // DOM 索引 ≠ Yoga 索引（因为有些 child 没有 yogaNode）
    let yogaIndex = 0;
    if (newChildNode.yogaNode && node.yogaNode) {
      for (let i = 0; i < index; i++) {
        if (node.childNodes[i]?.yogaNode) yogaIndex++;
      }
    }
    node.childNodes.splice(index, 0, newChildNode);
    if (newChildNode.yogaNode && node.yogaNode) {
      node.yogaNode.insertChild(newChildNode.yogaNode, yogaIndex);
    }
    markDirty(node);
  }
};
```

---

### 模式 3: Commit 后的同步时序保证

`resetAfterCommit` 按严格顺序触发两个回调:

```
resetAfterCommit(rootNode)
    ↓
  1. rootNode.onComputeLayout()  → yogaNode.calculateLayout(terminalColumns)
    ↓  （同步阻塞）
  2. rootNode.onRender()         → 调度帧渲染，消费 computed layout
```

这保证了 **消费布局数据** 一定在 **布局数据更新** 之后，类似于浏览器中 JS 同步阻塞读取布局信息的行为。

---

### 模式 4: 上下文感知的类型提升

通过 `HostContext.isInsideText` 追踪当前是否在文本节点内部:

```typescript
// src/ink/reconciler.ts:316-329
getChildHostContext(parentHostContext, type): HostContext {
  const isInsideText =
    type === 'ink-text' || type === 'ink-virtual-text' || type === 'ink-link';
  if (parentHostContext.isInsideText === isInsideText) {
    return parentHostContext;  // 复用对象，减少 GC
  }
  return { isInsideText };
},
```

当 `ink-text` 嵌套在另一个 `ink-text` 内时:
- 外层 `ink-text` → 正常创建（有 Yoga 节点）
- 内层 `ink-text` → 自动提升为 `ink-virtual-text`（无 Yoga 节点）

防止冗余的布局节点。

---

### 模式 5: Yoga 资源清理

移除节点时必须清理 Yoga 的 WASM 资源，且要在 `freeRecursive()` 之前清除引用:

```typescript
// src/ink/reconciler.ts:95-104
const cleanupYogaNode = (node: DOMElement | TextNode): void => {
  const yogaNode = node.yogaNode;
  if (yogaNode) {
    yogaNode.unsetMeasureFunc();
    // 先清引用，再释放——防止并发访问 freed WASM memory
    clearYogaNodeReferences(node);
    yogaNode.freeRecursive();
  }
};
```

---

## Ink DOM 节点类型一览

```
节点类型              有 Yoga 节点?   用途
─────────────────────────────────────────────
ink-root              ✅              根节点
ink-box               ✅              Flex 容器（类似 <div>）
ink-text              ✅              文本叶子节点（有 measureFunc）
ink-raw-ansi          ✅              预渲染 ANSI 字符串
ink-virtual-text      ❌              嵌套 text（纯文本拼接）
ink-link              ❌              OSC 8 超链接包装
ink-progress          ❌              进度条特殊节点
#text                 ❌              纯文本值
```

---

## 反模式

| 反模式 | 后果 | 正确做法 |
|--------|------|---------|
| 为每个 DOM 元素都创建 Yoga 节点 | 浪费内存和布局时间 | 按节点类型选择性创建 |
| 移除节点后不释放 Yoga 资源 | WASM 内存泄漏 | `cleanupYogaNode` 递归释放 |
| 释放后仍持有 yogaNode 引用 | 访问 freed memory 崩溃 | 先 `clearYogaNodeReferences` 再 `freeRecursive` |
| 每次 React commit 同步触发完整渲染 | 高频 commit 阻塞主线程 | 通过 `onRender` 节流调度 |
| 不区分 DOM 索引和 Yoga 索引 | `insertBefore` 位置错误 | 跳过无 Yoga 节点的 child 计算 |

---

## 适用场景

- 构建非浏览器 React 渲染目标（终端 UI、Canvas、自定义图形系统）
- 需要将 React 声明式模型接入自定义布局引擎
- 终端 TUI 框架开发
- 需要同时管理"逻辑节点树"和"布局节点树"两棵不同步的树

---

## 关键源文件索引

| 文件 | 核心内容 |
|------|---------|
| `src/ink/reconciler.ts` | 完整 HostConfig 实现 (512行) |
| `src/ink/dom.ts` | Ink DOM 节点类型、createNode、markDirty、树变异 |
| `src/ink/ink.tsx` | Ink 主类：串联 reconciler + renderer + terminal I/O |
| `src/ink/layout/node.ts` | Yoga LayoutNode 接口 |
| `src/ink/layout/engine.ts` | createLayoutNode 工厂 |
| `src/ink/styles.ts` | applyStyles：style → Yoga 属性映射 |
