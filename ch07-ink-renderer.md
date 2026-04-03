# 第 7 章：终端渲染引擎（Ink）

## 7.1 为什么用 React 渲染终端？

Claude Code 做了一个非常规的技术选择：用 **React** 来构建终端 UI。

传统终端 UI 库（如 blessed、inquirer）直接操作 ANSI 转义码，代码难以维护。Claude Code 的方案是：

```
React 组件树
    │
    ▼
自定义 React Reconciler（src/ink/reconciler.ts）
    │
    ▼
虚拟 DOM（ink-root, ink-box, ink-text 节点）
    │
    ▼
Yoga 布局引擎（Flexbox 计算）
    │
    ▼
Screen Buffer（字符网格）
    │
    ▼
ANSI 转义码输出到终端
```

这让开发者可以用熟悉的 React 模式编写终端 UI，享受组件复用、状态管理、热更新等好处。

---

## 7.2 渲染管道

[src/ink/render-to-screen.ts](../../src/ink/render-to-screen.ts) 是渲染管道的核心：

```typescript
// 共享的渲染资源（跨调用复用，减少分配开销）
let root: DOMElement | undefined;
let charPool: CharPool | undefined;
let hyperlinkPool: HyperlinkPool | undefined;
let output: Output | undefined;

export function renderToScreen(
  el: ReactElement,
  width: number,
): { screen: Screen; height: number } {
  // 1. 初始化根节点（首次调用）
  if (!root) {
    root = createNode('ink-root');
    root.focusManager = new FocusManager(() => false);
    stylePool = new StylePool();
    charPool = new CharPool();
    hyperlinkPool = new HyperlinkPool();
    container = reconciler.createContainer(root, LegacyRoot, ...);
  }
  
  // 2. React Reconciler 更新虚拟 DOM
  reconciler.updateContainer(el, container, null, null);
  
  // 3. Yoga 计算布局
  root.yogaNode.calculateLayout(width, undefined, DIRECTION_LTR);
  
  // 4. 渲染到 Screen Buffer
  const screen = createScreen(width, height);
  renderNodeToOutput(root, screen, ...);
  
  return { screen, height };
}
```

### 性能数据

注释中记录了实测性能：
- 每次调用约 1-3ms（Yoga 分配 + 布局计算 + 绘制）
- 使用 `LegacyRoot`（同步模式）而非 `ConcurrentRoot`，避免调度器跨根泄漏

---

## 7.3 Yoga 布局引擎

[src/native-ts/yoga.ts](../../src/native-ts/yoga.ts) 封装了 Facebook 的 Yoga 布局引擎——这是 React Native 使用的同一个 Flexbox 实现：

```typescript
// 每个 DOM 节点都有对应的 Yoga 节点
const yogaNode = Yoga.Node.create();
yogaNode.setFlexDirection(FLEX_DIRECTION_ROW);
yogaNode.setAlignItems(ALIGN_CENTER);
yogaNode.setWidth(80);

// 计算布局
yogaNode.calculateLayout(
  terminalWidth,
  undefined,  // 高度自适应
  DIRECTION_LTR
);

// 读取计算结果
const { left, top, width, height } = yogaNode.getComputedLayout();
```

这意味着 Claude Code 的终端 UI 支持完整的 Flexbox 布局，包括：
- `flexDirection: row | column`
- `alignItems: flex-start | center | flex-end`
- `justifyContent: space-between | space-around`
- `flexWrap: wrap`
- `padding`, `margin`, `border`

---

## 7.4 Screen Buffer

[src/ink/screen.ts](../../src/ink/screen.ts) 实现了字符网格缓冲区：

```typescript
// Screen 是一个二维字符网格
type Screen = {
  cells: Cell[][]  // [row][col]
  width: number
  height: number
}

type Cell = {
  char: string      // 字符内容
  styleId: number   // 样式 ID（颜色、粗体等）
  width: number     // 字符宽度（中文字符占 2 格）
}
```

### 对象池优化

为了减少 GC 压力，Screen 使用对象池：

```typescript
// 字符池：复用相同字符的对象
class CharPool {
  private pool = new Map<string, number>();
  intern(char: string): number { ... }
}

// 样式池：复用相同样式的对象
class StylePool {
  private pool = new Map<string, number>();
  intern(style: Style): number { ... }
}
```

---

## 7.5 键盘输入处理

[src/ink/parse-keypress.ts](../../src/ink/parse-keypress.ts) 解析终端键盘输入：

```typescript
// 终端键盘输入是原始字节序列
// 例如：方向键上 = ESC [ A = [0x1B, 0x5B, 0x41]

export function parseKeypress(data: Buffer): Keypress {
  const str = data.toString('utf8');
  
  // 特殊键序列映射
  if (str === '\x1B[A') return { key: 'upArrow' };
  if (str === '\x1B[B') return { key: 'downArrow' };
  if (str === '\x1B[C') return { key: 'rightArrow' };
  if (str === '\x1B[D') return { key: 'leftArrow' };
  if (str === '\x1B') return { key: 'escape' };
  if (str === '\r') return { key: 'return' };
  if (str === '\t') return { key: 'tab' };
  
  // Ctrl 组合键
  if (str === '\x03') return { key: 'c', ctrl: true };  // Ctrl+C
  if (str === '\x04') return { key: 'd', ctrl: true };  // Ctrl+D
  
  // 普通字符
  return { key: str };
}
```

---

## 7.6 焦点管理

[src/ink/focus.ts](../../src/ink/focus.ts) 实现了类似浏览器的焦点管理系统：

```typescript
class FocusManager {
  private focusedId: string | null = null;
  private focusableComponents: Map<string, FocusableComponent> = new Map();
  
  // Tab 键切换焦点
  focusNext(): void {
    const ids = [...this.focusableComponents.keys()];
    const currentIndex = ids.indexOf(this.focusedId ?? '');
    const nextIndex = (currentIndex + 1) % ids.length;
    this.focus(ids[nextIndex]);
  }
  
  // Shift+Tab 反向切换
  focusPrevious(): void { ... }
}
```

---

## 7.7 ANSI 颜色与样式

[src/ink/colorize.ts](../../src/ink/colorize.ts) 处理终端颜色输出：

```typescript
// 将样式对象转换为 ANSI 转义码
function applyStyle(text: string, style: Style): string {
  let result = text;
  
  if (style.color) {
    result = chalk[style.color](result);
  }
  if (style.backgroundColor) {
    result = chalk.bgHex(style.backgroundColor)(result);
  }
  if (style.bold) {
    result = chalk.bold(result);
  }
  if (style.italic) {
    result = chalk.italic(result);
  }
  
  return result;
}
```

---

## 小结

Claude Code 的终端渲染引擎是一个完整的 React → 终端渲染管道：
- React 组件树提供声明式 UI 开发体验
- Yoga 引擎提供完整的 Flexbox 布局支持
- Screen Buffer 提供高效的字符网格渲染
- 对象池减少 GC 压力，保证渲染性能

这个架构使得 Claude Code 的 UI 代码与 Web 开发高度相似，降低了维护成本。
