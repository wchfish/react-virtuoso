# AGENTS.md

这是给 Codex 和其他编码代理在 `react-virtuoso` 仓库中工作的项目说明。

阅读本文件时，先把它当成仓库定位手册：代码在哪里、采用哪条技术架构、改完后应该怎样验证。除非用户明确要求写入仓库文件，否则不要把计划、调查记录、报告、审计结果、执行笔记等内部工作产物放进仓库。

如果本文件和 `package.json`、`pnpm-workspace.yaml`、实际文件系统，或子目录里更具体的说明冲突，以当前仓库状态和更具体的说明为准。

## 高优先级规则

- 这是一个 `pnpm` workspaces monorepo。默认从仓库根目录运行命令，除非 package 本地上下文更清晰。
- 根目录 `pnpm lint` **不会**运行 TypeScript 类型检查。需要类型验证时单独运行 `pnpm typecheck`。
- 迭代时优先运行最窄范围的验证。只有改动跨 package 边界时，才优先使用根目录命令。
- 不要把 `data-testid` 当作样式 hook。它只保留给测试使用。样式或语义选择器使用 `data-table-element-role` 等语义属性。
- 不要为了内部工作产物创建 `plans/`、`prompts/`、`reports/` 等临时 markdown 目录。
- 如果用户要求 PRP、计划、调查、执行笔记、验证笔记、提示词、报告、审计或导出发现，除非用户明确要求仓库文件，否则使用共享 Notion 模板：`https://www.notion.so/3461834d8eff81318a4cccaf19b0de51`。
- 如果遇到用途不明确的现有 markdown 文件，删除或迁移前先询问用户。

## NPM Registry 查询

根目录 `package.json` 使用了 `devEngines.packageManager`，并设置 `"onFail": "error"`。在这个配置下，`pnpm info` 和 `pnpm view` 可能会因为委托给 npm 而失败，因为 npm 11 会拒绝该请求。

查询包版本时，从仓库外运行 npm：

```bash
(cd /tmp && npm info <package> version)
```

## 工作区地图

### 主要 Package

- `packages/react-virtuoso` - 主虚拟滚动库，支持列表、网格、表格、分组列表、feed 和 window scrolling。
- `packages/data-table` - 虚拟化 React 数据表 package，基于较新的 reactive engine package，目前是本仓库最活跃的 package。
- `packages/masonry` - 虚拟化 masonry 布局 package。
- `packages/gurx` - reactive state 库，被 masonry 和旧 Virtuoso 风格内部实现使用。
- `packages/reactive-engine-core` - 框架无关的 graph/node reactive engine。
- `packages/reactive-engine-react` - reactive engine 的 React bindings。
- `packages/reactive-engine-query`、`packages/reactive-engine-router`、`packages/reactive-engine-storage` - reactive engine 的可选集成。
- `packages/reactive-engine-examples` - reactive engine 系列示例。
- `packages/virtuoso-skills` - 分发给代理使用的 skills 源 package。

### 应用和示例

- `apps/virtuoso.dev` - Astro/Starlight 文档站点。
- `examples` - 独立 workspace，用于共享 Ladle/integration 示例。
- `packages/react-virtuoso/examples` - `packages/react-virtuoso/e2e` 使用的示例页面。
- `packages/data-table/src/_stories` - 用于 data table 开发和浏览器测试的 Ladle stories。

## 技术架构

### `react-virtuoso`

`packages/react-virtuoso` 是原始虚拟化实现。公共组件从 `src/index.tsx` 导出，内部围绕自定义 `urx` stream/state 系统组织。

重要入口：

- `src/Virtuoso.tsx` - list 和 grouped-list 组件表面。
- `src/VirtuosoGrid.tsx` - grid 虚拟化组件。
- `src/TableVirtuoso.tsx` - table 虚拟化组件。
- `src/component-interfaces/` - React 组件的公共 TypeScript 接口。
- `src/react-urx/` - 把 `urx` systems 连接到 React props、state 和 subscriptions 的桥接层。
- `src/urx/` - stream primitives、system composition、operators、actions 和 transformers。

核心 systems：

- `src/listSystem.ts` 组合主要 list 行为。
- `src/sizeSystem.ts`、`src/sizeRangeSystem.ts`、`src/totalListHeightSystem.ts` 跟踪已测量 item 尺寸、尺寸区间和总滚动高度。
- `src/listStateSystem.ts` 计算可见 items 和 list state。
- `src/domIOSystem.ts` 处理 DOM 测量和滚动交互。
- `src/scrollToIndexSystem.ts`、`src/scrollIntoViewSystem.ts`、`src/followOutputSystem.ts`、`src/windowScrollerSystem.ts` 处理滚动定位模式。
- `src/groupedListSystem.ts`、`src/topItemCountSystem.ts` 和 sticky 相关工具支持分组内容和固定顶部内容。

修改这个 package 时，遵循现有 `*System.ts` 和 `react-urx` 模式。除非任务明确是迁移，否则不要把新的 `reactive-engine-*` 模型混入这里。

### `data-table`

`packages/data-table` 是较新的 package，架构和 `react-virtuoso` 分开。它使用 `@virtuoso.dev/reactive-engine-core` 和 `@virtuoso.dev/reactive-engine-react` 编排状态，表格自身行为按 feature 和领域目录组织。

重要入口：

- `src/index.ts` - package exports。
- `src/core/VirtuosoDataTable.tsx` - 主 table 组件。
- `src/core/` - model bridge、rendering content、hooks、loading state 和 core actions。
- `src/model/` - local/remote data model state、actions、persistence 和 reserved actions。
- `src/columns/` - column definitions、header tree、size distribution、registry 和 header slots。
- `src/rows/` - row 和 group-header rendering/state。
- `src/layout/` - scroll layout root、virtualized content 和 scroller elements。
- `src/scroll/` - scroll state、callbacks、reverse scroll fix、smooth scroll 和 row navigation。
- `src/resize/`、`src/sizing/` - 已测量尺寸跟踪和 range math。
- `src/features/` - column reorder、resize、visibility、dynamic columns、state persistence 等可选 feature modules。

修改这个 package 时，把 feature 逻辑放到对应的 `src/features/*` 模块，把核心契约放到 `src/core`、`src/model`、`src/columns` 或 `src/rows`。优先复用现有 engine node/cell/stream 模式，不要从 `react-virtuoso` 的 `urx` internals 借实现。

### Reactive Engine Packages

`packages/reactive-engine-*` workspaces 提供 `data-table` 使用的底层 engine。

- `reactive-engine-core/src/Engine.ts`、`nodes.ts`、`combinators.ts`、`operators.ts` 和 `pipe.test.ts` 定义 graph、nodes、combinators 和 typed pipe 机制。
- `reactive-engine-react` 提供 engine-backed state 的 React 集成。
- integration packages 在 core engine 之上增加 query、router 和 storage 能力。

修改这些 package 可能影响 `data-table` 和未来的 engine 消费方。除了 package 本地测试，还要运行受影响下游 package 的检查，通常是 `packages/data-table`。

### 文档站点

`apps/virtuoso.dev` 是文档站点。package 文档会从源文档和 API metadata 生成到 app 中，因此修改文档时编辑 package 内的源文档，不要直接改生成后的 docs content。

## 命令

### 根目录

- `pnpm build` - 构建所有 workspaces。
- `pnpm typecheck` - 对所有 workspaces 运行类型检查。
- `pnpm lint` - 对所有 workspaces 运行 lint scripts。
- `pnpm format` - 使用 `oxfmt` 格式化，并通过 docs app 对 `.astro` 运行 Prettier。
- `pnpm format:check` - 检查格式。
- `pnpm test` - 运行 workspace test scripts。
- `pnpm e2e` - 运行 workspace e2e scripts。
- `pnpm lint:md` / `pnpm lint:md:fix` - markdown lint / fix。
- `pnpm ci` - 完整 CI：setup、format check、build、typecheck、lint、markdown lint、test、e2e。
- `pnpm dev:docs` - 启动文档站点。
- `pnpm build:skills` - 重新生成 skill references 和公开 skill mirrors。
- `pnpm validate:skills` - 校验 Claude 和 Codex plugin manifests。
- `pnpm changeset-add` - 添加 changeset。

### `packages/react-virtuoso`

- `pnpm build` - Vite build。
- `pnpm typecheck` - `tsgo --noEmit`。
- `pnpm lint` - `oxlint --type-aware --type-check`。
- `pnpm test` - Vitest。
- `pnpm test:watch` - Vitest watch mode。
- `pnpm e2e` - Playwright tests。
- `pnpm ladle` - 在 Ladle 中预览 package examples。

### `packages/data-table`

- `pnpm build` - `tsc && vite build`。
- `pnpm typecheck` - `tsgo -b --noEmit`。
- `pnpm lint` - `oxlint --type-aware --type-check`。
- `pnpm test` - `vitest run --browser.headless`。
- `pnpm test:all` - 包含 slow browser tests。
- `pnpm check` - `format:check + lint + typecheck`。
- `pnpm dev` - 启动 Ladle serve mode。
- `pnpm dev:build` / `pnpm dev:preview` - Ladle build / preview。

### `apps/virtuoso.dev`

- `pnpm dev` - Astro dev server。
- `pnpm build` - `shadcn build && astro build`。
- `pnpm lint` - `oxlint --type-aware --type-check && astro check`。
- `pnpm format` / `pnpm format:check` - `oxfmt` 加 `.astro` 的 Prettier。

## 文档工作流

不要直接编辑这些生成目录：

- `apps/virtuoso.dev/src/content/docs/data-table/`
- `apps/virtuoso.dev/src/content/docs/react-virtuoso/`
- `apps/virtuoso.dev/src/content/docs/masonry/`
- `apps/virtuoso.dev/src/content/docs/gurx/`
- `apps/virtuoso.dev/src/content/docs/message-list/`

改源文档：

- `packages/data-table/README.md` 和 `packages/data-table/docs/*.md`
- `packages/react-virtuoso/README.md` 和 `packages/react-virtuoso/docs/*.md`
- `packages/masonry/README.md`
- `packages/gurx/README.md`
- 修改 engine package 文档时，改 `packages/reactive-engine-*/docs/*.md`。

产品文档变更后：

- 从仓库根目录运行 `pnpm lint:md`。
- 如果改了 docs-site content、Astro components、registry code 或 docs integration code，运行 `pnpm --filter @virtuoso.dev/virtuoso.dev lint`。

## Plugin 分发

本仓库以三种形式分发 `virtuoso-skills`：

- Claude Code plugin：`packages/virtuoso-skills/`
- Codex plugin：`plugins/virtuoso-skills/`
- `npx skills` 使用的根目录 mirror：`skills/`

编辑源 skill 文件：`packages/virtuoso-skills/skills/<name>/SKILL.md`。

不要直接编辑这些生成输出：

- `packages/virtuoso-skills/skills/*/references/`
- `plugins/virtuoso-skills/skills/`
- `skills/`

用下面命令重新生成：

```bash
pnpm build:skills
```

合并后，公开 Codex plugin 安装命令：

```bash
codex plugin marketplace add petyosi/react-virtuoso --ref main --sparse .agents/plugins --sparse plugins/virtuoso-skills
codex plugin add virtuoso-skills@virtuoso
```

不要提交 `.agents/skills/` 给 Codex/OpenCode/Cursor，除非跨代理安装方案发生变化。该目标路径由 `npx skills` 管理。

## 验证指南

选择和改动范围匹配的最窄验证：

- `packages/react-virtuoso` 代码变更：在 `packages/react-virtuoso` 中运行 `pnpm lint && pnpm typecheck && pnpm test`；如果修改了滚动行为、DOM 测量、examples 或浏览器行为，再运行 `pnpm e2e`。
- `packages/data-table` 代码变更：在 `packages/data-table` 中运行 `pnpm check && pnpm test`；如果修改了 sizing、scrolling、resize、reorder、persistence 或 browser-only 行为，再运行 `pnpm test:all`。
- `packages/reactive-engine-*` 变更：在对应 package 中运行 `pnpm lint && pnpm typecheck && pnpm test`；同时运行受影响下游 package 的检查，通常是 `packages/data-table`。
- 只改 markdown 文档：运行 `pnpm lint:md`。
- docs site 或 registry 变更：运行 `pnpm --filter @virtuoso.dev/virtuoso.dev lint`。
- skill 变更：运行 `pnpm build:skills && pnpm validate:skills`。
- 跨 package 的大范围变更：从根目录运行 `pnpm typecheck && pnpm lint && pnpm test`。

## 代码风格

- TypeScript 是默认语言。保持强类型，除非周围代码已经要求，否则避免使用 `any`。
- 引入新抽象前，先匹配现有模块边界。
- 尺寸计算、滚动计算、state nodes、stream operators 和 DOM 测量优先复用现有工具，不要重复实现。
- React 组件保持 function component 和 hooks 风格，并匹配本地模式。
- 格式化由 `oxfmt` 处理：single quotes、无 semicolons，并使用仓库配置的行宽。
- pre-commit hooks 由 `lefthook` 管理。只有明确的 WIP commit 才使用：

```bash
LEFTHOOK=0 git commit -m "WIP: ..."
```
