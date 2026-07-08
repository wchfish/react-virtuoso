# react-virtuoso 边界场景处理逻辑分析

> 基于 `packages/react-virtuoso/src/` 源码的工程分析，覆盖 22 个 system 文件、3 个组件文件、4 个 hook 文件及 8 个工具文件。

---

## 一、空状态 / 冷启动（6 层防御）

| 边界场景 | 处理方式 | 位置 |
|----------|---------|------|
| `totalCount === 0` | 直接返回 `EMPTY_LIST_STATE`，不进入计算管线 | `listStateSystem.ts:248` |
| 视口尚未测量（尺寸全为 0） | 用 `initialItemCount` 预渲染少量元素触发首轮测量 | `listStateSystem.ts:252` |
| sizeTree 为空（无任何测量数据） | 构造 `probeItemSet` 发送探测元素，等待 ResizeObserver 回报 | `listStateSystem.ts:258` |
| `offsetTree` 为空时查询 offset | `offsetOf()` 直接返回 0 | `sizeSystem.ts:114` |
| 列表容器无子元素 | `getChangedChildSizes` 返回 `null`，不上报空尺寸 | `useChangedChildSizes.ts:99` |
| 列表为空时渲染 `EmptyPlaceholder` | 检测 `totalCount === 0`，渲染用户自定义占位组件 | `Virtuoso.tsx:118` |

---

## 二、数据变更（增量更新而非全量重建）

| 边界场景 | 处理方式 | 位置 |
|----------|---------|------|
| `data` 与 `totalCount` 异步不同步 | `dataChangeInProgress` 检测，跳过中间态的重计算 | `listStateSystem.ts:217` |
| `totalCount` 缩小（删除了尾部元素） | 用最后一个已知尺寸覆盖尾部区域，清除过期测量（#896） | `sizeSystem.ts:360` |
| `firstItemIndex` 变化（头部增/删） | 检测偏移量 → `unshiftWith`（前插）/ `shiftWith`（前删），触发增量重构 | `sizeSystem.ts:514` |
| 分组模式下 unshift 导致第一组扩容 | 检测 `prependedGroupItemsCount !== unshiftWith`，移除旧的组内尺寸 | `sizeSystem.ts:576` |
| 分组模式下 shift 后 sizeTree 为空 | 直接返回原 sizes，不崩溃 | `sizeSystem.ts:649` |
| shift 后索引可能为负 | 所有 key 用 `Math.max(0, k + shiftWith)` 夹紧 | `sizeSystem.ts:669` |

---

## 三、滚动边界（防止越界与死循环）

| 边界场景 | 处理方式 | 位置 |
|----------|---------|------|
| iOS 橡皮筋效果产生负 `scrollTop` | `Math.max(scrollTop, 0)` | `useScrollTop.ts:76` |
| 程序化滚动目标超出范围 | 夹紧到 `[0, maxScrollTop]` | `useScrollTop.ts:141` |
| 滚动容器尺寸为 0（未挂载） | 提前返回，不执行滚动 | `useScrollTop.ts:103` |
| 内容高度 ≤ 视口高度（无滚动条） | 直接标记完成，不执行滚动 | `useScrollTop.ts:143` |
| `scrollToIndex` 到末项 + `align: 'end'` | 额外加上 footer 高度 | `scrollToIndexSystem.ts:81` |
| `scrollToIndex` 偏移量太小不触发重渲染 | 1.2 秒超时清理订阅，防止挂起 | `scrollToIndexSystem.ts:113` |
| `scrollToIndex` 期间列表内容变化（如图片加载） | 订阅 `listRefresh`，检测到变化则重试 | `scrollToIndexSystem.ts:98` |
| 滚动到底部的浮点误差 | `isAtBottom` 使用 4px 容差阈值 | `stateFlagsSystem.ts:101` |
| Smooth scroll 永不抵达目标（浏览器不触发 scroll 事件） | 1 秒超时强制标记完成 | `useScrollTop.ts:157` |
| Smooth scroll 已在边界 | `scrollTop === 0` 或 `scrollTop === max` 时立即标记完成 | `useScrollTop.ts:74` |

---

## 四、尺寸测量（防御性 + 合并优化）

| 边界场景 | 处理方式 | 位置 |
|----------|---------|------|
| 元素尺寸为 0（未挂载/隐藏） | 打印 ERROR 日志，不崩溃 | `useChangedChildSizes.ts:113` |
| 相邻元素同尺寸 | 自动合并为一个区间 `{start, end, value}`，减少上报量 | `useChangedChildSizes.ts:117` |
| 尺寸未变化 | 与 `dataset.knownSize` 比较，跳过未变项 | `useChangedChildSizes.ts:115` |
| 浏览器不支持 ResizeObserver | observer 为 `null`，只依赖初始 mount 回调 | `useSize.ts:16` |
| 元素已脱离 DOM 时 ResizeObserver 回调 | 检查 `offsetParent !== null`，跳过已移除元素 | `useSize.ts:19` |
| 插入的区间与已有区间完全重叠 | `rangeIncludes` 检测，跳过插入 | `sizeSystem.ts:72` |
| 新区间与邻居部分重叠（需要拆分/合并） | 前邻居若同值则 remove → 合并；后邻居若超出则插入新边界点 | `sizeSystem.ts:80-95` |
| 分组模式首次探测返回双区间（组+项不同尺寸） | 特殊分支分别插入组尺寸和项尺寸 | `sizeSystem.ts:155` |
| `heightEstimates` 提供的逐项预估值 | 仅在 sizeTree 为空时生效，自动合并连续同值项 | `sizeSystem.ts:415` |

---

## 五、初始渲染时序（避免闪烁和位置跳动）

| 边界场景 | 处理方式 | 位置 |
|----------|---------|------|
| 系统流在组件挂载前就开始发射 | `didMount` 仅当 `propsReady` 为 `true` 后发射，所有计算管线的前置过滤器 | `propsReadySystem.ts:7` |
| 非零 `initialTopMostItemIndex` 但尚未滚动到位 | `scrolledToInitialItem = false`，`listStateSystem` 返回空列表 | `initialTopMostItemIndexSystem.ts:19` |
| 初始滚动时尺寸未知 | 过滤 `!empty(sizeTree) \|\| isDefined(defaultItemSize)`，等 4 帧布局后再滚动 | `initialTopMostItemIndexSystem.ts:41` |
| 初始滚动完成前出现内容闪烁 | 容器设置 `visibility: 'hidden'`，直到 `initialItemFinalLocationReached` | `Virtuoso.tsx:116` |
| `initialScrollTop` 需要等元素渲染完毕 | 过滤 `listState.items.length > 1` 后再执行滚动 | `initialScrollTopSystem.ts:10` |
| Grid 恢复状态时短暂显示旧内容 | `stateRestoreInProgress` 期间返回 `null` | `VirtuosoGrid.tsx:124` |

---

## 六、分组列表（组头/项的坐标系转换）

| 边界场景 | 处理方式 | 位置 |
|----------|---------|------|
| 视觉索引与数据索引的转换 | `originalIndexFromItemIndex` 累加组头偏移量 | `sizeSystem.ts:120` |
| 粘性组头检测 | `findMaxKeyValue(groupOffsetTree, scrollTop - headerHeight)` 找当前顶部的组 | `groupedListSystem.ts:37` |
| 内部平铺索引 → 分组展示索引 | `transposeItems` 逐项判定是否命中组边界，赋予 `type: 'group'` 或 `groupIndex` | `listStateSystem.ts:131` |
| 组数量变化时重建 sizeTree | 当 `fixedGroupSize` + `defaultItemSize` 均已知时，按最新组数量重建 | `sizeSystem.ts:446` |
| `position: sticky` 兼容旧浏览器 | `positionStickyCssValue()` 特性检测 → 回退 `-webkit-sticky` | `utils/positionStickyCssValue.ts` |

---

## 七、Chat/Feed 类场景（followOutput）

| 边界场景 | 处理方式 | 位置 |
|----------|---------|------|
| 初始化时不应触发跟随 | `skip(1)` 跳过首次 `totalCount` 发射 | `followOutputSystem.ts:56` |
| 变高项尚未测量，滚动到末项位置可能错误 | 若无 `fixedItemSize`，等 `listRefresh` 后再滚动 | `followOutputSystem.ts:79` |
| 新跟随请求到达时上一个未完成 | 取消 `pendingScrollHandle`，避免竞态 | `followOutputSystem.ts:72` |
| 用户在底部但上方内容高度增加（如图片加载）推离底部 | 检测 `notAtBottomBecause === 'SIZE_INCREASED'` → 自动滚回底部 | `followOutputSystem.ts:98` |
| 视口缩小（如移动端键盘弹出） | 检测 `VIEWPORT_HEIGHT_DECREASING` → 自动滚回底部 | `followOutputSystem.ts:136` |
| content 刷新但 totalCount 不变 | `scan` 检测前后值相同 → 标记 `refreshed`，激活增量陷阱 | `followOutputSystem.ts:113` |
| `followOutput` 接受多种值类型 | `normalizeFollowOutput` 将 `false`/`true`/`'smooth'`/`'auto'` 统一为内部格式 | `followOutputSystem.ts:18` |

---

## 八、布局多样性（水平/RTL/Window Scroller/Gap）

| 边界场景 | 处理方式 | 位置 |
|----------|---------|------|
| 水平方向 | 样式轴互换（`overflowX`/`overflowY`），测量轴切换（`offsetWidth`），item 设为 `inline-block` | `Virtuoso.tsx:207` |
| RTL 文本方向 | `getLogicalScrollLeft` / `getPhysicalScrollLeft` 双向转换，`WeakMap` 缓存方向 | `utils/horizontalScroll.ts` |
| Window Scroller | `scrollTop = Math.max(0, windowScrollTop - offsetTop)` 坐标变换 | `windowScrollerSystem.ts:17` |
| 自定义滚动父容器 | 视口矩形基于该容器而非 window 计算 | `useWindowViewportRect.ts:22` |
| CSS Gap 为 `normal` | 返回 0 | `useChangedChildSizes.ts:127` |
| CSS Gap 非 px 单位 | 打印 WARN 日志 | `useChangedChildSizes.ts:125` |
| 内容短于视口 + `alignToBottom` | `paddingTopAddition = max(0, viewportHeight - totalListHeight)`，容器 `flex-direction: column` | `alignToBottomSystem.ts:9` |
| Table 变体的 align-to-bottom | 渲染 `<FillerRow height={paddingTop}>` 代替 CSS padding（表格布局限制） | `TableVirtuoso.tsx:237` |

---

## 九、Props 防御（类型联合 + 默认值）

| 边界场景 | 处理方式 | 位置 |
|----------|---------|------|
| `firstItemIndex < 0` | ERROR 日志 + 引导文案 | `sizeSystem.ts:532` |
| `data` 为 `undefined` | `data \|\| []` 兜底，`data?.[i]` 可选链 | `listStateSystem.ts:256` |
| 未传 `totalCount` | 从 `data.length` 自动推导 | `listStateSystem.ts:477` |
| `scrollToIndex` 参数不完整 | `normalizeIndexLocation`：对齐默认 `start`，行为默认 `auto`，偏移默认 0 | `scrollToIndexSystem.ts:15` |
| `minOverscanItemCount` 可为数字或 `{bottom, top}` | 运行时类型判断分发 | `listStateSystem.ts:170` |
| `overscan` / `increaseViewportBy` 的数字/对象联合 | 方向感知分发（UP→TOP用main，DOWN→BOTTOM用main） | `sizeRangeSystem.ts:21` |
| 自定义组件是 DOM 标签字符串 | `contextPropIfNotDomElement` 拦截，避免 React `context` 作为 DOM 属性报错 | `Virtuoso.tsx:237` |

---

## 十、性能防护（避免重复计算 + React 并发安全）

| 边界场景 | 处理方式 | 位置 |
|----------|---------|------|
| 尺寸变化和滚动事件同时触发重计算 | `recalcInProgress` 守护，重建期间阻塞 listState 管线 | `listStateSystem.ts:216` |
| `data` 和 `totalCount` 分别发射触发双重计算 | `dataChangeInProgress` 检测，等两者同步后再算 | `listStateSystem.ts:217` |
| 流值未变化被重复发射 | `duc`（distinctUntilChanged）包裹所有进入计算管线的流 | 全局使用 |
| ResizeObserver 同步回调中多余的 rAF | `skipAnimationFrameInResizeObserver` 标志允许同步执行 | `useSize.ts:22` |
| 同步多次 publish 触发重复计算 | `throttleTime(0)` 合并为微任务级别 | `followOutputSystem.ts:165` |
| 滚动停止检测 | `debounceTime(100)` 100ms 无事件才标记 `isScrolling: false` | `stateFlagsSystem.ts:61` |
| 浮点数比较 | `approximatelyEqual(a, b)`，容差 1.01px | `useScrollTop.ts:143` |
| 父组件重渲染导致子组件不必要渲染 | 所有 `Header`/`Footer`/`Items`/`Scroller` 包裹 `React.memo` | 全局组件 |
| React 18 并发模式撕裂 | `useSyncExternalStore`（React 18+），降级 `useState` + `useLayoutEffect` | `react-urx/index.tsx:282` |
| 程序化滚动修正被误判为用户滚动方向 | `isScrollingBy` 标志 → `scrollDirection` 保持上一方向不变 | `stateFlagsSystem.ts:192` |

---

## 工程原则总结

react-virtuoso 的边界处理可以归纳为几个核心工程原则：

1. **渐进测量（Progressive Measurement）** — 不假设所有元素都能一次性测完，用"探针→测量→外推→修正"闭环逐步收敛，盲区用最后已知尺寸估算
2. **增量更新（Incremental Update）** — 数据变更时不重建整个 sizeTree，而是做 diff + 局部 insert/remove，最小化计算量
3. **防御先行（Defense in Depth）** — 每个入口都有 nil/empty/out-of-range 检查，不会因缺数据崩溃；负值夹紧、空树返回零值、已脱离 DOM 跳过
4. **时序控制（Temporal Ordering）** — `didMount`、`recalcInProgress`、`dataChangeInProgress` 三道阀门防止中间态进入重计算管线
5. **去重消抖（Deduplication）** — `duc`、`distinctUntilChanged`、`throttleTime`、`debounceTime` 层层过滤，确保只有"真正需要计算"的状态变化才触发昂贵操作
6. **优雅降级（Graceful Degradation）** — 无 ResizeObserver 退化为初始回调、无 `scrollBehavior` CSS 退化为 `auto`、无 React 18 退化为 `useState`+`useLayoutEffect`
