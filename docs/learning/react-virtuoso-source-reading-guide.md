# React-Virtuoso 源码阅读顺序

## 阶段一：urx 响应式原语（地基）

> **目标**：理解流是什么、系统如何组合。不涉及任何虚拟滚动概念。

### 1.1 核心概念（按顺序阅读）

| 顺序 | 文件 | 行数 | 阅读重点 |
|------|------|------|---------|
| 1 | `src/urx/constants.ts` | ~10 | 理解 `PUBLISH`、`SUBSCRIBE`、`RESET`、`VALUE` 四个动作码，这是整个 urx 的协议基础 |
| 2 | `src/urx/streams.ts` | ~130 | `stream()` 和 `statefulStream()` 的实现。注意两者在 `SUBSCRIBE` 动作上的差异——stateful 立即回调 |
| 3 | `src/urx/actions.ts` | ~150 | `publish()`、`subscribe()`、`connect()`、`getValue()`、`handleNext()`。理解 `connect` = `subscribe(source, publish(target))` |
| 4 | `src/urx/pipe.ts` | ~200 | `pipe()` 和操作符：`map`、`filter`、`scan`、`debounceTime`、`throttleTime`、`withLatestFrom`、`distinctUntilChanged`、`duc`（即 `distinctUntilChanged` 缩写） |
| 5 | `src/urx/transformers.ts` | ~80 | `combineLatest()` 和 `merge()`。理解 `combineLatest` 返回的是 stateful stream |

### 1.2 系统组合

| 顺序 | 文件 | 行数 | 阅读重点 |
|------|------|------|---------|
| 6 | `src/urx/system.ts` | ~220 | `system()`、`tup()`、`init()`。**`init()` 只有 10 行**，理解 DFS 后序遍历 + singleton 机制 |
| 7 | `src/urx/utils.ts` | ~100 | `tup()` 只是 `return args`（类型辅助）。`tap()`、`noop()`、`curry2to1()` 等工具函数 |

### 1.3 动手验证

```bash
cd packages/react-virtuoso
# 阅读 urx 相关测试
cat test/urx/*.test.ts
```

---

## 阶段二：AA 树（尺寸存储的数据结构）

> **目标**：理解稀疏尺寸存储的核心数据结构，不涉及任何 React 或滚动逻辑。

| 顺序 | 文件 | 行数 | 阅读重点 |
|------|------|------|---------|
| 1 | `src/AATree.ts` | ~290 | 自平衡二叉搜索树。重点看 `insert`、`find`、`findMaxKeyValue`、`ranges`、`rangesWithin`、`walk`、`arrayToRanges`。`split`/`skew` 平衡操作可先跳过 |
| 2 | `src/utils/binaryArraySearch.ts` | ~40 | `findIndexOfClosestSmallerOrEqual`、`findClosestSmallerOrEqual`、`findRange`。offsetTree 二分查找的基础 |

**核心理解**：AA 树存储 `{k: index, v: height}` 的边界点，`ranges()` 将节点列表转为 `[{start, end, value}]` 区间表示。

---

## 阶段三：最小完整示例 — 理解 system 模式

> **目标**：通过最简单的系统理解"系统定义 → 流连接 → 输出"的完整模式。

| 顺序 | 文件 | 行数 | 阅读重点 |
|------|------|------|---------|
| 1 | `src/loggerSystem.ts` | ~30 | 最简单的系统：只有一个 `log` 流，没有依赖 |
| 2 | `src/propsReadySystem.ts` | ~15 | 两个流的系统：`didMount` + `propsReady` |
| 3 | `src/contextSystem.ts` | ~10 | 单流系统，理解 props → stream 的模式 |
| 4 | `src/domIOSystem.ts` | ~80 | **第一个有实际意义的系统**。创建 scrollTop、viewportHeight、deviation 等流。理解 singleton 模式（`{ singleton: true }`） |

**阅读技巧**：每个系统遵循相同模板：

```typescript
export const xxxSystem = u.system(
  ([dependency1, dependency2]) => {  // 解构依赖
    const input1 = u.stream()         // 创建输入流
    const output1 = u.statefulStream(0) // 创建输出流

    u.connect(/* 流A */, /* 流B */)  // 连接流

    return { input1, output1 }        // 返回流字典
  },
  u.tup(depSystem1, depSystem2)       // 声明依赖
)
```

---

## 阶段四：核心虚拟化管线（核心中的核心）

> **目标**：理解"滚动位置 → 可见像素区间 → 可见 item 列表"的完整数据流。

### 4.1 尺寸系统

| 顺序 | 文件 | 行数 | 阅读重点 |
|------|------|------|---------|
| 1 | `src/sizeSystem.ts` | ~600 | **最大、最重要的文件**。包含 `insertRanges`、`createOffsetTree`、`sizeStateReducer`、`offsetOf`、`rangesWithinOffsets`。先看完前 250 行（到 `createOffsetTree`），后面是系统定义和分组支持 |
| 2 | `src/sizeRangeSystem.ts` | ~120 | `visibleRange` 的计算。重点理解方向判断逻辑和 overscan 的方向性 |

### 4.2 状态系统

| 顺序 | 文件 | 行数 | 阅读重点 |
|------|------|------|---------|
| 3 | `src/stateFlagsSystem.ts` | ~280 | `isScrolling`、`scrollDirection`、`isAtBottom`、`atBottomState`、`scrollVelocity`、`lastJumpDueToItemResize` |
| 4 | `src/listStateSystem.ts` | ~470 | `listState` 的完整计算。重点看 `buildListState`、`rangesWithinOffsets` 的使用、overscan 扩展、`endReached`/`startReached`/`rangeChanged` 的产生逻辑 |

### 4.3 高度系统

| 顺序 | 文件 | 行数 | 阅读重点 |
|------|------|------|---------|
| 5 | `src/totalListHeightSystem.ts` | ~35 | 一行公式：`footerHeight + fixedFooterHeight + headerHeight + fixedHeaderHeight + offsetBottom + bottom` |

---

## 阶段五：React 桥接层

> **目标**：理解 urx 流如何映射为 React 组件的 props、events、methods。

| 顺序 | 文件 | 行数 | 阅读重点 |
|------|------|------|---------|
| 1 | `src/react-urx/index.tsx` | ~280 | `systemToComponent()` 函数。理解 `applyPropsToSystem`（props → stream publish）、`buildEventHandlers`（stream → event callback）、`buildMethods`（stream → imperative method）、`useEmitterValue`（`useSyncExternalStore` + stream） |
| 2 | `src/Virtuoso.tsx` | ~570 | Virtuoso 组件定义。看 `listComponentPropsSystem`（组件级 props 流）和 `combinedSystem`（listSystem + propsSystem 的合并）。`Items` 子组件的 `React.memo` 包装 |

---

## 阶段六：功能系统（按复杂度排序）

> **目标**：理解各个功能如何作为独立系统插拔到核心管线。

| 顺序 | 文件 | 行数 | 复杂度 | 阅读重点 |
|------|------|------|--------|---------|
| 1 | `src/topItemCountSystem.ts` | ~30 | ⭐ | 固定顶部 item。最简功能系统 |
| 2 | `src/initialItemCountSystem.ts` | ~25 | ⭐ | SSR 初始渲染数量 |
| 3 | `src/initialTopMostItemIndexSystem.ts` | ~40 | ⭐ | 初始滚动位置 |
| 4 | `src/initialScrollTopSystem.ts` | ~25 | ⭐ | 初始 scrollTop |
| 5 | `src/alignToBottomSystem.ts` | ~50 | ⭐⭐ | 内容不足视口时底部对齐 |
| 6 | `src/scrollToIndexSystem.ts` | ~130 | ⭐⭐ | 编程式滚动到指定 index。`align`、`offset`、behavior 的处理 |
| 7 | `src/followOutputSystem.ts` | ~200 | ⭐⭐⭐ | 自动跟底。`handleNext(listRefresh, ...)` 的时序处理、`trapNextSizeIncrease` 的二次修正 |
| 8 | `src/upwardScrollFixSystem.ts` | ~120 | ⭐⭐⭐ | 向上滚动时的偏差补偿。Mobile Safari 的特殊处理 |
| 9 | `src/windowScrollerSystem.ts` | ~100 | ⭐⭐ | Window 滚动模式 |
| 10 | `src/scrollSeekSystem.ts` | ~35 | ⭐ | Seek 模式的状态切换 |
| 11 | `src/groupedListSystem.ts` | ~200 | ⭐⭐⭐ | 分组列表。groupIndices 计算、sticky header |
| 12 | `src/scrollIntoViewSystem.ts` | ~90 | ⭐⭐ | 将指定 item 滚动到可见区域 |
| 13 | `src/stateLoadSystem.ts` | ~80 | ⭐⭐ | 状态保存/恢复 |

---

## 阶段七：顶层组装

| 顺序 | 文件 | 行数 | 阅读重点 |
|------|------|------|---------|
| 1 | `src/listSystem.ts` | ~180 | **所有系统的顶层组合者**。看 `featureGroup1System`（内层 11 个系统 → 1 个）和 `listSystem`（外层 11 个系统）。理解系统分层的动机：减少构造函数的参数数量 |
| 2 | `src/index.tsx` | ~15 | 公开导出：Virtuoso、GroupedVirtuoso、VirtuosoGrid、TableVirtuoso 及类型 |

---

## 阅读进度建议

```
          urx 原语           ← 1-2天（阶段一）
             │
          AA 树              ← 0.5天（阶段二）
             │
          简单系统            ← 0.5天（阶段三）
             │
    ┌───────┼───────┐
    │       │       │
  sizeSystem  sizeRange  stateFlags  ← 2-3天（阶段四：核心管线）
    │       │       │
    └───┬───┴───┬───┘
        │       │
    listState   totalListHeight
        │
  react-urx + Virtuoso.tsx  ← 1-2天（阶段五：桥接层）
        │
   各功能系统（7-8个）       ← 3-4天（阶段六）
        │
    listSystem.ts           ← 0.5天（阶段七）
```

**总计约 8-12 天**。建议每天聚焦一个阶段，边读边在测试文件中打断点验证理解。核心时间投资在**阶段四**（核心管线）和**阶段六**（功能系统）。

---

## 辅助阅读材料

### 补充文件

| 文件 | 用途 |
|------|------|
| `src/interfaces.ts` | 公共类型定义，按需查阅 |
| `src/component-interfaces/Virtuoso.ts` | VirtuosoProps、VirtuosoHandle 类型 |
| `src/component-interfaces/VirtuosoGrid.ts` | VirtuosoGrid 接口 |
| `src/component-interfaces/TableVirtuoso.ts` | TableVirtuoso 接口 |
| `src/utils/correctItemSize.ts` | 修正 box-sizing 影响的 item 尺寸 |
| `src/utils/context.ts` | VirtuosoMockContext 测试工具 |
| `src/comparators.ts` | `tupleComparator`、`rangeComparator` 等比较函数 |
| `src/hooks/` | React hooks（按需查阅） |

### 测试文件

```bash
packages/react-virtuoso/test/
├── urx/                  # urx 流操作符的单元测试
├── sizeSystem.test.ts    # 尺寸系统测试
├── listStateSystem.test.ts
├── AATree.test.ts
└── ...
```

### 示例文件

```bash
packages/react-virtuoso/examples/   # Ladle 示例
packages/react-virtuoso/e2e/        # Playwright E2E 测试
```
