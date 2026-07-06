# React-Virtuoso 学习路线图

## 项目概览

这是一个 **pnpm workspaces monorepo**，包含两套并行的响应式状态管理系统：

| 维度 | urx（旧） | reactive-engine（新） |
|---|---|---|
| 使用者 | `react-virtuoso`、`masonry` | `data-table` |
| 模式 | 函数即流（function-as-stream） | 符号节点图（Symbol-based node graph） |
| 编排 | `system()` 组合 | `Engine` 类 + `EngineProvider` |

---

## 第一阶段：前置知识（1-2天）

### 1.1 响应式编程基础

- **概念**：流（stream）、发射（emit）、订阅（subscribe）、操作符（operator）
- **推荐阅读**：RxJS 入门教程（理解 `map`、`filter`、`scan`、`combineLatest`、`debounceTime` 等操作符）
- **不需要精通 RxJS**，只需理解这些操作符的语义

### 1.2 虚拟滚动原理

- **核心问题**：10万条数据，DOM 只能渲染可见区域的几百条
- **关键概念**：
  - 总高度估算 → 滚动条比例
  - 滚动位置 → 计算可见索引范围
  - 仅渲染可见项 + 上下 overscan
  - 用 padding/transform 占位不可见区域

### 1.3 动手验证

```bash
cd packages/react-virtuoso
pnpm ladle  # 启动示例浏览器，查看各种 demo
```

浏览 `examples/` 目录下的示例，理解各组件的使用方式和使用场景。

---

## 第二阶段：urx 响应式内核（3-5天）

> **目标**：理解 react-virtuoso 的自研状态管理引擎

### 2.1 核心文件

```
packages/react-virtuoso/src/urx/
├── streams.ts       # stream()、statefulStream()
├── pipe.ts          # pipe()、map()、filter()、scan()、debounceTime()
├── actions.ts       # publish()、subscribe()、connect()
├── transformers.ts  # combineLatest()、merge()、duc()
├── system.ts        # system()、init()、tup()
└── utils.ts         # tap()、noop()、compose()
```

### 2.2 学习路径

**Step 1 — 理解流的基本概念**

- `stream<T>()` 返回一个函数，该函数接受 `PUBLISH | SUBSCRIBE | RESET` 动作码
- `statefulStream<T>(initialValue)` 额外支持 `VALUE` 动作，记住最后一次值
- 阅读 `streams.ts`（约 60 行），理解 emitter/subscription 闭包模式

**Step 2 — 操作符链**

- `pipe(stream, operator1, operator2, ...)` — 创建变换链
- `map(fn)`、`filter(predicate)`、`scan(reducer, seed)` — 标准操作符
- `withLatestFrom(depot, project)` — 类似 RxJS 同名操作符
- `debounceTime(ms)`、`throttleTime(ms)` — 时间控制

**Step 3 — 系统组合**

- `system(fn, dependencies?)` — 定义一组流及其依赖
- `tup(sys1, sys2, ...)` — 声明系统依赖元组
- `init(systemSpec)` — 递归初始化系统树，建立订阅链
- `connect(source, sink)` — 将一个流的输出连接到另一个流

**Step 4 — 变换器**

- `combineLatest([stream1, stream2, ...])` — 合并多个流的最新值，返回 statefulStream
- `duc(stream)` — 去重（distinct until changed）

### 2.3 动手实践

阅读测试文件了解用法：

```bash
# 查看 urx 相关测试
ls packages/react-virtuoso/test/
```

建议手动写一个 mini 响应式系统，实现 `stream`、`statefulStream`、`pipe`、`map`、`combineLatest`。

---

## 第三阶段：核心虚拟化引擎（5-7天）

> **目标**：理解数据如何变成屏幕上的 DOM 元素

### 3.1 数据流概览

```
scrollTop (DOM事件)
    ↓
domIOSystem.ts        ← 测量 viewportHeight、scrollTop、scrollHeight
    ↓
sizeRangeSystem.ts    ← 计算可见像素区间 [startOffset, endOffset]
    ↓
sizeSystem.ts         ← AATree 维护 item 尺寸区间
    ↓
listStateSystem.ts    ← 像素区间 → 可见 item 列表 [{index, offset, size}]
    ↓
Virtuoso.tsx          ← 渲染 ListItem → 调用 itemContent(index)
```

### 3.2 核心文件学习顺序

| 序号 | 文件 | 核心职责 | 关键概念 |
|------|------|---------|---------|
| 1 | `AATree.ts` | 增强区间树 | `insert()`、`find()`、`walk()`、区间合并 |
| 2 | `sizeSystem.ts` | item 尺寸管理 | `sizeTree`、`offsetTree`、`sizeRanges`、ResizeObserver 回调 |
| 3 | `domIOSystem.ts` | DOM交互层 | `scrollTop`、`viewportHeight`、`deviation`、`scrollTo`/`scrollBy` |
| 4 | `sizeRangeSystem.ts` | 可见范围计算 | `startOffset`、`endOffset`、overscan |
| 5 | `listStateSystem.ts` | 可见item列表 | `listState` → `{items, topListHeight, offsetTop, ...}` |
| 6 | `totalListHeightSystem.ts` | 总高度 | `totalListHeight` = 已测量高度 + 未测量估算 |

### 3.3 关键算法理解

**AATree（增强区间树）**：

- 每个 item 的尺寸记录为 `{startIndex, endIndex, size}`
- 相同高度的连续 item 自动合并为区间
- `offsetOf(index)` 通过二分查找 + 树遍历定位 item 位置
- 这是实现**可变高度虚拟滚动**的核心数据结构

**尺寸测量流程**：

1. item 挂载到 DOM
2. ResizeObserver 回调 → `sizeRanges` 流
3. `sizeSystem` 更新 `sizeTree` 和 `offsetTree`
4. 触发 `listStateSystem` 重新计算可见范围
5. React 重新渲染受影响的 item

### 3.4 动手实践

```bash
cd packages/react-virtuoso
pnpm test  # 运行单元测试，观察测试用例
# 重点查看 sizeSystem 和 listStateSystem 的测试
```

---

## 第四阶段：功能系统（3-5天）

> **目标**：理解各个功能系统如何插拔到核心引擎

### 4.1 系统依赖图

```
listSystem（顶层组合者）
├── sizeSystem              ← 尺寸跟踪
├── domIOSystem             ← DOM 交互（单例）
├── listStateSystem         ← 可见状态计算
├── groupedListSystem       ← 分组 + 粘性头
├── scrollToIndexSystem     ← 编程式滚动
├── followOutputSystem      ← 聊天/Feed 自动跟底
├── windowScrollerSystem    ← window 滚动模式
├── scrollSeekSystem        ← 快速滚动占位
├── topItemCountSystem      ← 固定顶部 item
├── alignToBottomSystem     ← 底部对齐
├── upwardScrollFixSystem   ← 向上滚动修复
├── initialTopMostItemIndexSystem ← 初始滚动位置
├── stateFlagsSystem        ← 滚动方向/边界标记
└── stateLoadSystem         ← 状态保存/恢复
```

### 4.2 重点功能系统

| 系统 | 学习价值 | 核心逻辑 |
|------|---------|---------|
| `groupedListSystem` | ⭐⭐⭐ | groupIndices 计算、sticky header 偏移 |
| `followOutputSystem` | ⭐⭐⭐ | 自动跟底 vs 用户手动滚动的判断逻辑 |
| `scrollToIndexSystem` | ⭐⭐ | align/offset 计算、smooth scroll |
| `windowScrollerSystem` | ⭐⭐ | 用 window 作为滚动容器，非 overflow:scroll |
| `scrollSeekSystem` | ⭐ | 快速滚动时用占位符替代真实渲染 |

### 4.3 学习技巧

- 每个系统都遵循相同模式：`system(() => { ... return { output1, output2 } })`
- 关注 `connect()` 调用——它们揭示系统间的数据流
- 画出系统间的连线图，可视化数据流向

---

## 第五阶段：React 桥接层（2-3天）

> **目标**：理解 urx 流如何映射到 React 组件

### 5.1 核心文件

```
packages/react-virtuoso/src/
├── react-urx/index.tsx     # systemToComponent() — 核心桥接函数
├── Virtuoso.tsx            # 列表组件实例
├── VirtuosoGrid.tsx        # 网格组件实例
└── TableVirtuoso.tsx       # 表格组件实例
```

### 5.2 `systemToComponent()` 的工作原理

```typescript
const { Component, useEmitterValue, useEmitter, usePublisher } = 
  systemToComponent(systemSpec, {
    required: { totalCount: 'totalCount' },   // 必需 props → 流
    optional: { itemContent: 'itemContent' }, // 可选 props → 流
    events: { rangeChanged: 'rangeChanged' }, // 回调 → 流订阅
    methods: { scrollToIndex: 'scrollToIndex' }, // 命令式方法 → 流发布
  })
```

返回的 `Component` 是一个 `React.forwardRef` 组件，内部：

1. 用 `useState` 惰性初始化 `init(systemSpec)`
2. 用 `useSyncExternalStore` 订阅状态流的值变化
3. 用 `useImperativeHandle` 暴露命令式方法

### 5.3 组件渲染层次

```
<Virtuoso>
  <VirtuosoScroller>          ← 滚动容器
    <VirtuosoTopItemList>     ← 固定顶部内容
    <VirtuosoList>            ← 实际渲染容器
      <VirtuosoFillerRow />   ← 顶部占位 (paddingTop/transform)
      {visibleItems.map(item =>
        <VirtuosoItem>        ← 每个可见 item
          {itemContent(index)}
        </VirtuosoItem>
      )}
      <VirtuosoFillerRow />   ← 底部占位
    </VirtuosoList>
  </VirtuosoScroller>
</Virtuoso>
```

### 5.4 动手实践

1. 阅读 `Virtuoso.tsx` — 理解 `combinedSystem` 如何组合 `listSystem` + `listComponentPropsSystem`
2. 对比 `VirtuosoGrid.tsx` — 看 grid 模式如何复用不同的 system
3. 调试：在关键 `useEmitterValue` 调用处打断点，观察数据流

---

## 第六阶段：新一代架构（3-5天）

> **目标**：理解 reactive-engine 如何替代 urx，以及 data-table 如何工作

### 6.1 reactive-engine-core

```
packages/reactive-engine-core/src/
├── Engine.ts        # 引擎核心：图管理、发布/订阅、状态存储
├── nodes.ts         # Cell、Stream、Trigger、Resource、DerivedCell
├── operators.ts     # map、filter、scan、debounceTime 等
├── combinators.ts   # combine、merge、pipe、link
└── pipe.test.ts     # 管道测试（最好的学习材料）
```

**与 urx 的关键差异**：

- urx: 流是**函数**，通过函数调用进行发布/订阅
- reactive-engine: 节点是 **Symbol 引用**，通过 `Engine.pub(symbol, value)` 发布

### 6.2 reactive-engine-react

```typescript
// 使用模式
<EngineProvider initFn={registerNodes}>
  <VirtuosoDataTable />
</EngineProvider>

// 组件内部
const scrollTop = useCellValue(scrollTop$)
const publish = usePublisher(someStream$)
```

### 6.3 data-table 架构

```
VirtuosoDataTable
├── EngineProvider (initFn 注册所有 Cell/Stream)
├── TableLayoutRoot
│   ├── column headers (sticky)
│   └── VirtualizedTableContent
│       ├── top filler
│       ├── visible rows → <Row> → <Cell>
│       └── bottom filler
├── features/
│   ├── column-reorder/     ← 拖拽排序
│   ├── column-resize/      ← 列宽调整
│   ├── column-visibility/  ← 列显隐
│   └── state-persistence/  ← 状态持久化
└── model/
    ├── local-model.ts      ← 本地数据
    └── remote-model.ts     ← 远程分页数据
```

**学习顺序**：

1. `core/VirtuosoDataTable.tsx` — 主组件
2. `sizing/` — 与 react-virtuoso 的 sizeSystem 对比
3. `layout/VirtualizedTableContent.tsx` — 渲染核心
4. `columns/` — 列定义系统
5. 选一个 `features/` 深入研究（推荐 `column-resize`）

### 6.4 动手实践

```bash
cd packages/data-table
pnpm dev  # 启动 Ladle stories
pnpm test # 运行测试
```

---

## 第七阶段：周边 Package（1-2天）

| Package | 关注点 | 学习价值 |
|---------|--------|---------|
| `gurx` | urx 的变体（Symbol 引用 + Realm） | 理解 urx 和 reactive-engine 之间的演变 |
| `masonry` | Pinterest 瀑布流布局 | 特殊布局算法的虚拟化实现 |
| `message-list` | 聊天消息列表 | 看 `followOutput` 的实际应用 |
| `reactive-engine-query` | URL 参数同步 | 理解 engine 扩展模式 |
| `reactive-engine-storage` | 持久化 | 理解 engine 扩展模式 |

---

## 第八阶段：综合实践（持续）

### 8.1 建议项目

1. **用 react-virtuoso 实现一个聊天应用** — 练习 `followOutput`、`atBottomStateChange`、`initialTopMostItemIndex`
2. **实现一个带分组的大列表** — 练习 `GroupedVirtuoso`、sticky headers、`groupCounts`
3. **给 data-table 添加一个新 feature** — 比如行选择、行拖拽排序
4. **尝试用 reactive-engine 构建一个新组件** — 体验新架构的开发体验

### 8.2 调试技巧

- urx 系统：在 `src/loggerSystem.ts` 中启用 `LogLevel.DEBUG`
- data-table：使用 React DevTools 观察 `EngineProvider` 上下文
- 在测试文件中 `it.only()` 隔离单个测试用例
- 使用 `pnpm ladle` 可视化调试组件行为

### 8.3 贡献指南

```bash
# 修改代码后的验证流程
cd packages/react-virtuoso
pnpm format && pnpm lint && pnpm typecheck && pnpm test
# 涉及 UI/滚动行为还需
pnpm e2e
```

---

## 总结：学习时间估算

| 阶段 | 内容 | 预计时间 |
|------|------|---------|
| 一 | 前置知识 | 1-2天 |
| 二 | urx 响应式内核 | 3-5天 |
| 三 | 核心虚拟化引擎 | 5-7天 |
| 四 | 功能系统 | 3-5天 |
| 五 | React 桥接层 | 2-3天 |
| 六 | 新一代架构 | 3-5天 |
| 七 | 周边 Package | 1-2天 |
| **合计** | | **18-29天** |

建议按顺序学习，**第二阶段和第三阶段是核心**，投入最多时间。理解 urx 系统和 AATree 尺寸跟踪后，其余部分都是在此基础上的具体应用和扩展。
