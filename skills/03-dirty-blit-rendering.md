# Dirty-Flag + Blit 增量渲染优化

## 解决什么问题

在流式 AI 界面中，每帧更新时屏幕上大部分内容不变（只有最新 token、spinner、或工具状态变化）。如果每帧都重绘整个屏幕缓冲区再做全量 diff，计算量是 O(rows × cols)——对于大屏幕和高频更新（60fps），这会成为瓶颈。

Claude Code 通过 **脏标记(dirty flag) + 块拷贝(blit)** 将每帧渲染降低到 O(变化的 cell 数量)。

---

## 核心模式

### 模式 1: Ink DOM 脏标记系统

每个 `DOMElement` 有一个 `dirty: boolean` 字段。当属性/样式/子节点发生变化时，`markDirty()` 向上冒泡设 `dirty = true`:

```typescript
// src/ink/dom.ts:393-413
export const markDirty = (node?: DOMNode): void => {
  let current: DOMNode | undefined = node;
  let markedYoga = false;

  while (current) {
    if (current.nodeName !== '#text') {
      (current as DOMElement).dirty = true;

      // 只对第一个遇到的文本叶子节点触发 Yoga 的 markDirty
      // （昂贵操作：触发重新测量文本换行）
      if (!markedYoga &&
          (current.nodeName === 'ink-text' || current.nodeName === 'ink-raw-ansi') &&
          current.yogaNode) {
        current.yogaNode.markDirty();
        markedYoga = true;
      }
    }
    current = current.parentNode;
  }
};
```

**关键设计**: `markedYoga = true` 后不再对祖先节点触发 `yogaNode.markDirty()`——因为 Yoga 的 dirty 会自动向上传播，只需在**最近的叶子**标记一次。

---

### 模式 2: 激进的脏避免

三道防线阻止不必要的 `markDirty()` 调用:

**防线 1: 跳过 `children` 属性**
```typescript
// src/ink/dom.ts:247-257
export const setAttribute = (node, key, value): void => {
  // React 每次 render 都传新的 children 引用
  // 如果当 attribute 处理，会导致每次都 markDirty
  if (key === 'children') return;
  if (node.attributes[key] === value) return;  // 值相等也跳过
  node.attributes[key] = value;
  markDirty(node);
};
```

**防线 2: style 做浅比较**
```typescript
// src/ink/dom.ts:266-274
export const setStyle = (node, style): void => {
  // React 经常每次 render 都 style={{...}} 创建新对象
  // 通过浅比较避免无意义的 markDirty
  if (stylesEqual(node.style, style)) return;
  node.style = style;
  markDirty(node);
};
```

**防线 3: textStyles 同理**
```typescript
// src/ink/dom.ts:276-289
export const setTextStyles = (node, textStyles): void => {
  if (shallowEqual(node.textStyles, textStyles)) return;
  node.textStyles = textStyles;
  markDirty(node);
};
```

---

### 模式 3: Blit 决策——块拷贝复用上一帧

渲染时，对每个节点检查是否可以直接拷贝上一帧的像素:

```typescript
// src/ink/render-node-to-output.ts:452-482 (概念简化)
const cached = nodeCache.get(node);
if (!node.dirty                           // 节点没有变化
    && !skipSelfBlit                       // 非强制重绘
    && node.pendingScrollDelta === undefined // 无待处理滚动
    && cached                              // 有上一帧缓存
    && cached.x === x && cached.y === y    // 位置没变
    && cached.width === width              // 尺寸没变
    && cached.height === height
    && prevScreen) {                       // 上一帧可用
  // ✅ 快路径: 直接块拷贝上一帧的矩形区域
  output.blit(prevScreen, fx, fy, fw, fh);
  return;  // 完全跳过子树遍历！
}
```

**效果**:
- Frame 0: 全量渲染 → Screen Buffer A
- Frame 1: 只有 spinner 变化 → 99% 的节点命中 blit，只重绘 spinner 区域
- diff 阶段: 只比较变化的 cell

---

### 模式 4: 双缓冲 + charCache 复用

渲染器维护 ping-pong 双缓冲，复用 `charCache`（已分词+字形聚类的行表示）:

```
Frame N:   [frontFrame]  ← 当前显示
           [backFrame]   ← 正在合成

Frame N+1: [backFrame]   ← 现在是当前显示
           [frontFrame]  ← 变成合成目标
```

`Output` 实例在帧间复用，`charCache`（Map<行内容, 已聚类结果>）避免对不变的行重复做 ANSI 分词和字形聚类。缓存每 5 分钟清空一次防止无限增长。

---

### 模式 5: 屏幕 Diff + 最小 Patch

`LogUpdate` 对 prevScreen 和 nextScreen 逐 cell 比较，只对变化的 cell 生成终端控制序列:

```
diff(prevScreen, nextScreen)
    ↓
  对每个变化的 cell:
    cursor-move(x, y) + style-transition(fromId, toId) + char-write
    ↓
  合并为最小 patch 序列
    ↓
  一次或少数几次 stdout.write()
```

---

### 模式 6: 污染追踪

当 prevScreen 被后渲染操作修改（如选区覆盖层 inverse）时，blit 会拷贝错误的内容。通过 `prevFrameContaminated` 标记禁用一帧的 blit:

```
Frame N: 正常渲染
  → 选区覆盖层修改了 screen buffer (inverse cell styles)
  → prevFrameContaminated = true

Frame N+1: blit 被禁用，全量渲染
  → prevFrameContaminated = false

Frame N+2: 恢复 blit 快路径
```

类似场景还有: alt-screen 缓冲区被重置、绝对定位节点被移除（可能覆盖了非兄弟节点的区域）。

---

### 模式 7: Damage Bounds

`layoutShifted` 标记用于检测布局偏移:

```typescript
// 概念: render-node-to-output.ts
let layoutShifted = false;

// 当任何节点的位置/尺寸与缓存不一致时
if (cached && (cached.x !== x || cached.y !== y || ...)) {
  layoutShifted = true;
}

export function didLayoutShift(): boolean { return layoutShifted; }
```

如果发生布局偏移（如 spinner 出现导致内容下移），触发 full-damage backstop——将整个屏幕标记为需要 diff 的区域。否则只 diff 有 damage 的区域。

---

## 优化效果对比

```
场景: 40行 x 120列 终端，流式输出 token

无优化:
  每帧: 渲染 4800 cells → diff 4800 cells → 输出所有 patch
  复杂度: O(4800) per frame

有 dirty+blit:
  每帧: blit 4750 cells (O(1) memcpy) + 渲染 50 cells → diff 50 cells
  复杂度: O(50) per frame

提升: ~96× 减少渲染计算
```

---

## 反模式

| 反模式 | 后果 | 正确做法 |
|--------|------|---------|
| 每次 prop 变更不做值比较直接标脏 | 每帧都全量重绘 | `setAttribute`/`setStyle` 做值相等/浅比较 |
| 非文本属性变更也触发 `yogaNode.markDirty()` | 不必要的文本重测量 | 只在 ink-text/ink-raw-ansi 叶子上触发 |
| 每帧分配新 screen buffer | GC 压力大，无法复用 charCache | 双缓冲 ping-pong 复用 |
| 不追踪 prevScreen 是否被污染 | blit 拷贝到错误内容 | `prevFrameContaminated` 标记 |
| diff 整个屏幕不考虑 damage bounds | 浪费计算 | `didLayoutShift()` 控制 diff 范围 |

---

## 适用场景

- 高频部分更新的终端 UI（流式文本、动画、进度条）
- 大屏幕终端需要控制每帧计算量
- 需要 60fps 流畅更新的 TUI 应用
- 任何需要"虚拟缓冲区 + 最小差异更新"的渲染系统

---

## 关键源文件索引

| 文件 | 核心内容 |
|------|---------|
| `src/ink/dom.ts:393-413` | `markDirty()` 实现、dirty 字段 |
| `src/ink/dom.ts:247-289` | setAttribute/setStyle/setTextStyles 中的脏避免 |
| `src/ink/render-node-to-output.ts:452-482` | blit 决策逻辑、nodeCache rect 追踪 |
| `src/ink/renderer.ts` | `createRenderer` 双缓冲、Output 复用 |
| `src/ink/output.ts` | 操作收集(write/blit/clear/shift)、charCache |
| `src/ink/screen.ts` | CharPool/StylePool 池化 |
| `src/ink/log-update.ts` | 屏幕 diff 算法 `diffEach` |
