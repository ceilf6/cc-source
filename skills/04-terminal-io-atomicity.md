# 终端 I/O 原子性与防闪烁

## 解决什么问题

终端模拟器是**逐字节**渲染收到的输出的。当一次复杂的 UI 更新需要多个转义序列和文字写入时，用户会在中间看到不完整的状态——空白区域、光标跳动、样式泄漏、内容闪烁。

Claude Code 通过 **6 个互补的技术** 确保终端输出的视觉原子性。

---

## 核心模式

### 模式 1: DEC 2026 同步输出 (BSU/ESU)

当终端支持 DEC Private Mode 2026 时，所有帧 patch 被包裹在 Begin/End Synchronized Update 之间:

```
\x1b[?2026h      ← BSU (Begin Synchronized Update)
  [光标移动]
  [样式变更]
  [文字写入]
  [清除区域]
  ...所有帧 patch...
\x1b[?2026l      ← ESU (End Synchronized Update)
```

终端在收到 BSU 后缓冲所有输出，收到 ESU 后一次性渲染——**用户只看到最终结果，看不到中间态**。

```typescript
// src/ink/ink.tsx — 帧输出包裹
// Alt-screen 帧前导
optimized.unshift(CURSOR_HOME_PATCH);  // CSI H 锚定到 (0,0)
if (this.needsEraseBeforePaint) {
  optimized.unshift(ERASE_THEN_HOME_PATCH);  // 原子: 2J + H 在 BSU/ESU 内
}

// Alt-screen 帧后导
optimized.push(this.altScreenParkPatch);  // 光标停到底部（prompt 位置）

// 包裹同步输出块（如果 DEC 2026 支持）
// BSU: \x1b[?2026h / ESU: \x1b[?2026l
```

**降级策略**: tmux 可能破坏原子性（拆分到多个 pane），此时跳过 BSU/ESU 避免无效开销。

---

### 模式 2: 节流渲染调度

通过 `throttle` 控制渲染频率，避免每次 React commit 都立即触发终端 I/O:

```typescript
// src/ink/ink.tsx:204-217
// 16ms = ~60fps，leading + trailing
this.scheduleRender = throttle(deferredRender, FRAME_INTERVAL_MS, {
  leading: true,   // 首次 commit 立即渲染（响应性）
  trailing: true,  // 窗口内末次 commit 也必触发（不丢更新）
});
```

```
时间线:
  0ms: commit1 → 立即渲染 (leading)
  5ms: commit2 → 跳过（在 16ms 窗口内）
  10ms: commit3 → 跳过
  16ms: → 渲染 commit3 的结果 (trailing)
```

效果: 一次流式 token 更新可能触发多次 React commit，但终端最多 60fps 更新。

---

### 模式 3: 微任务延迟渲染

`deferredRender` 使用 `queueMicrotask` 而非直接调用:

```typescript
const deferredRender = () => queueMicrotask(this.onRender);
```

**为什么?** React 的 `resetAfterCommit` 在 commit 阶段执行，此时 refs 和 `useLayoutEffect` 已运行但 DOM 可能还有后续更新。`queueMicrotask` 确保:

1. 所有同步的 React 工作完成
2. `useLayoutEffect` 中的状态更新已提交
3. `useDeclaredCursor` 的光标位置声明已就绪

在同一个事件循环 tick 内执行，吞吐量不受影响。

---

### 模式 4: 延迟擦除——resize 不闪白

终端 resize 时，如果立即 `ERASE_SCREEN`，用户会看到 ~80ms 的白屏（等待新帧渲染）。

```
❌ 传统方式:
  resize事件 → ERASE_SCREEN → [80ms 白屏] → 渲染新帧

✅ Claude Code 方式:
  resize事件 → 设 needsEraseBeforePaint = true → [旧内容保持显示]
  下一帧渲染时 → BSU + ERASE + 新内容 + ESU → 原子替换
```

```typescript
// 概念代码
onResize() {
  this.needsEraseBeforePaint = true;  // 标记，不立即擦
  this.scheduleRender();               // 触发新帧
}

onRender() {
  if (this.needsEraseBeforePaint) {
    // 擦除被包在 BSU/ESU 内，和新内容一起原子提交
    optimized.unshift(ERASE_THEN_HOME_PATCH);
    this.needsEraseBeforePaint = false;
  }
}
```

---

### 模式 5: DECSTBM 硬件滚动

当 ScrollBox 内容滚动时，不逐行重写整个区域，而是利用终端的 Set Scroll Region 能力:

```
DECSTBM 滚动:
  CSI top;bottom r     ← 设置滚动区域
  CSI n S  / CSI n T   ← 向上/下滚动 n 行
  CSI r                ← 重置滚动区域

终端硬件移动行内容，只需重绘新露出的行
```

```typescript
// src/ink/log-update.ts — ScrollHint 处理
if (altScreen && next.scrollHint && decstbmSafe) {
  const { top, bottom, delta } = next.scrollHint;
  // 使用 DECSTBM + SU/SD 硬件滚动
  shiftRows(prev.screen, top, bottom, delta);
  scrollPatch = [DECSTBM + SU_or_SD + RESET_SCROLL_REGION + CURSOR_HOME];
}
```

---

### 模式 6: 字符/样式/超链接池化

将字符串比较降级为整数比较，大幅加速 cell-level diff:

```typescript
// src/ink/screen.ts — CharPool
class CharPool {
  private ascii: Int32Array = initCharAscii();  // charCode → index

  intern(char: string): number {
    if (char.length === 1 && code < 128) {
      const cached = this.ascii[code];
      if (cached !== -1) return cached;  // ASCII 快路径: O(1) 数组查找
    }
    return this.stringMap.get(char) ?? this.add(char);
  }
}

// StylePool
class StylePool {
  intern(styles: AnsiCode[]): number { ... }

  // 预缓存的样式转换字符串，按 (fromId, toId) 对缓存
  // warmup 后零分配
  transition(fromId: number, toId: number): string { ... }
}
```

**Cell diff 对比**:
```
无池化: strcmp("Hello") + strcmp("\x1b[31m") + strcmp("http://...") = O(n) 字符串比较
有池化: int(42) !== int(42) = O(1) 整数比较
```

---

### 模式 7: Alt-Screen 光标锚定

Alt-screen 模式下，每帧开始时强制将光标重置到 (0,0):

```typescript
// src/ink/ink.tsx:569-585
let prevFrame = this.frontFrame;
if (this.altScreenActive) {
  // 重置光标到 (0,0)，所有光标移动基于此计算
  prevFrame = { ...this.frontFrame, cursor: ALT_SCREEN_ANCHOR_CURSOR };
}
```

**自愈效果**: 如果外部程序（如通知弹窗、tmux 面板切换）移动了光标位置，下一帧会自动从 (0,0) 重新计算所有位置——无需检测或恢复。

---

## 技术总览图

```
React commit
    ↓
  resetAfterCommit
    ↓
  onComputeLayout (Yoga)
    ↓
  scheduleRender ← throttle(16ms, leading+trailing)
    ↓
  queueMicrotask(onRender)
    ↓
  renderNodeToOutput (dirty+blit 优化)
    ↓
  LogUpdate.render (screen diff → patch)
    ↓
  writeDiffToTerminal
    ↓
  ┌─ BSU ────────────────────────┐
  │ cursor-home                  │
  │ [erase if needsEraseBeforePaint] │
  │ [DECSTBM scroll if hint]    │
  │ patch1: move + style + char  │
  │ patch2: move + style + char  │
  │ ...                          │
  │ cursor-park (bottom)         │
  └─ ESU ────────────────────────┘
       ↓
  终端原子渲染
```

---

## 反模式

| 反模式 | 后果 | 正确做法 |
|--------|------|---------|
| 多次 `stdout.write()` 不做 BSU/ESU 包裹 | 用户看到中间态闪烁 | 检测 DEC 2026 支持并包裹 |
| resize 时同步 ERASE_SCREEN | ~80ms 白屏闪烁 | 设 `needsEraseBeforePaint`，在下一帧原子擦除 |
| 每次 React commit 都同步渲染到终端 | 终端 I/O 过载，帧率异常 | `throttle(16ms)` 节流 |
| cell diff 用字符串比较 | O(n) per cell | `CharPool`/`StylePool` 池化降为 O(1) |
| 不处理外部光标偏移 | 累积位置错误 | Alt-screen 锚定 CSI H 自愈 |

---

## 适用场景

- 终端 TUI 需要流畅的流式文本输出（无闪烁）
- 高频更新的终端 UI（动画、进度、实时数据）
- 需要处理 terminal resize 而不闪白
- 跨终端兼容性（tmux、iTerm2、Windows Terminal）
- 需要优化终端 I/O 性能的场景

---

## 关键源文件索引

| 文件 | 核心内容 |
|------|---------|
| `src/ink/ink.tsx` | scheduleRender 节流+微任务、BSU/ESU 包裹、needsEraseBeforePaint |
| `src/ink/log-update.ts` | writeDiffToTerminal、DECSTBM 滚动、screen diff 算法 |
| `src/ink/screen.ts` | CharPool/StylePool/HyperlinkPool 池化实现 |
| `src/ink/terminal.ts` | SYNC_OUTPUT_SUPPORTED 能力检测 |
| `src/ink/constants.ts` | FRAME_INTERVAL_MS = 16 |
| `src/ink/renderer.ts` | 双缓冲 + Output 复用 |
