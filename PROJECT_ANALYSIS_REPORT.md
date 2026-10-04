# Apache ECharts 6.1.0 项目分析报告

> 分析对象：`D:\dev\echarts`（git `master`，HEAD `34f4d927e`，`package.json` version `6.1.0`）
> 方法：静态源码通读 + 目录/行数统计 + 构建与测试脚本、CI 配置核对。所有结论均附相对路径与行号，便于复核。
> 说明：
> 1. 工作区未执行 `npm install`（无 `node_modules`），报告中的构建/测试结论来自脚本与配置阅读，非运行验证。
> 2. 工作区另有一批未跟踪的 `src/**/*.doc.md`（12 个）与 `.workbuddy/`，非本次分析生成，本报告未把它们作为结论依据。
> 3. 行号基于当前 HEAD 快照，若后续合入新提交可能偏移。

---

## 0. 摘要

Apache ECharts 是一个**基于 zrender 的声明式可视化框架**，而不是"一组画图函数"。它的核心价值在于：把用户写的一份 `option` 声明，经过**预处理 → 组件合并 → 数据建模 → 数据加工 → 坐标系创建 → 视觉编码 → 布局 → 渲染**这条固定的流水线，编译成 zrender 的图形元素树，并在数据或交互变化时以**增量、渐进、可动画**的方式复用既有元素。

从工程规模看：

| 指标 | 数值 |
|---|---|
| `src/*.ts` | 596 个文件，约 138,500 行 |
| 图表类型目录 `src/chart/` | 23 个（22 个功能目录 + `helper`），约 34,500 行 |
| 组件目录 `src/component/` | 25 个（23 个功能 + `axis`/`helper`），约 36,700 行 |
| 坐标系目录 `src/coord/` | 8 个（cartesian/polar/geo/radar/parallel/single/calendar/matrix），约 17,000 行 |
| 数据层 `src/data/` | 约 8,800 行（含 `SeriesData` 1321 行、`DataStore` 1156 行） |
| 注册型扩展点调用 | `registerAction` 54 处、`registerLayout` 28 处、`registerVisual` 23 处、`registerProcessor` 17 处、`registerCoordinateSystem` 10 处 |
| 运行时依赖 | 仅 `zrender@6.1.0` 与 `tslib@2.3.0` |
| 测试资产 | Jest 单测 26 个 spec；HTML 用例 606 个；交互录制 273 份；视觉基准标记 243 份 |
| 随包分发 | 36 个主题、27 种语言 |

一句话概括架构：**"声明式 option + 单例式注册表 + 基于 Task 流水线的增量渲染 + 可插拔的图表/组件/坐标系扩展"**。

---

## 1. 项目定位与依赖关系

### 1.1 定位

- 面向浏览器的**交互式图表库**（`package.json:4`），Canvas / SVG 双渲染后端，支持 SSR。
- TypeScript first：源码全量 TS，发布时生成 ESM / CJS / UMD 三套类型入口（见 `package.README.md`）。
- Apache 顶级项目：ASF license header 强制（`build/checkHeader.js`）、`NOTICE`、源码发布与重建校验（`.github/workflows/source-release.yml`）。

### 1.2 依赖极简性

`package.json:75-78` 中运行时依赖只有两项：

```json
"dependencies": { "tslib": "2.3.0", "zrender": "6.1.0" }
```

但源码对 zrender 的耦合**很深**：`src` 中 import 了 **61 个不同的** `zrender/src/*` 内部模块（`core/util`、`core/env`、`core/Eventful`、`core/BoundingRect`、`graphic/*`、`canvas/Painter`、`svg/Painter`、`animation/Animator`、`contain/*` 等）。这意味着：

- 优点：包体小、无第三方运行时负担，zrender 作为"图形内核"被充分复用（事件、动画、脏矩形、SVG 导出、路径命中检测等全部下沉）。
- 代价：**内核与框架版本强锁**（`src/core/echarts.ts:152-154` 导出 `dependencies = { zrender: '6.1.0' }`），zrender 的内部重构会直接击穿 ECharts；这也是仓库 wiki 要求用 `node_modules/zrender` 符号链接联调的原因。

---

## 2. 仓库结构与模块地图

```
echarts/
├── src/                       # 唯一事实源
│   ├── core/                  # 引擎：ECharts 实例、Scheduler、task、ExtensionAPI、CoordinateSystem、lifecycle、locale
│   ├── model/                 # 模型层：Global、Component、Series、Model、OptionManager
│   ├── view/                  # 视图基类：Chart（含 renderTask）、Component
│   ├── chart/                 # 23 类图表（含 helper）
│   ├── component/             # 25 类组件（含 axis/helper）
│   ├── coord/                 # 8 类坐标系 + Axis/Scale 适配 + View（漫游变换）
│   ├── scale/                 # interval/log/ordinal/time + break（断轴）
│   ├── data/                  # Source、SeriesData、DataStore、Graph、Tree、transform、DataDiffer
│   ├── visual/                # 视觉编码：style、symbol、VisualMapping、aria、decal、tokens
│   ├── layout/                # 布局算法：barGrid、barPolar、points、barCommon
│   ├── animation/             # universalTransition、customGraphicTransition、morph
│   ├── label/                 # LabelManager、labelLayout、labelStyle、sectorLabel
│   ├── processor/             # 内置数据处理器：dataStack、dataSample、dataFilter、negativeDataFilter
│   ├── renderer/              # canvas / svg 安装器
│   ├── preprocessor/          # backwardCompat（历史 option 兼容）
│   ├── legacy/                # 旧行动作、getTextRect
│   ├── i18n/                  # 27 种语言
│   ├── theme/                 # 内置 dark 主题
│   ├── export/                # 公共导出面：core/api/charts/components/features/option/renderers/all
│   └── echarts*.ts            # 4 个打包入口（all / common / simple / blank）
├── extension-src/             # bmap、dataTool（独立打包）
├── ssr/client/src/            # SSR 客户端 hydrate
├── build/                     # rollup / tsc / i18n / header / dev-fast / nightly / source-release
├── test/                      # ut（Jest）、runTest（视觉回归）、types（d.ts 测试）、node、606 个 *.html
├── theme/  i18n/  dist/       # 打包产物（部分入库）
└── package.json               # exports 子路径地图（约 127 条）
```

**分层不是靠目录栈，而是靠依赖方向约束**：

```
        echarts*.ts (入口，副作用注册)
             │
   ┌─────────┴──────────┐
   │  chart / component │  ← 只依赖 model/view/coord/data/util
   └─────────┬──────────┘
   core (Scheduler/ExtensionAPI/CoordinateSystemManager)
             │
   model (Global/Series/Component/OptionManager)
             │
   data / coord / scale / visual / layout / label / animation
             │
   util (types/model/graphic/number/clazz/states/...)  +  zrender
```

`src/core/echarts.ts` 是全仓的"汇聚点"：它同时 import 了 model、view、coord、visual、processor、animation、legacy、i18n 等 30+ 模块（`src/core/echarts.ts:19-144`），所以它既是引擎核心，也是**内置能力的注册中心**（文件末尾 `3360-3428` 行注册了 seriesStyleTask、dataStack、loading、highlight/downplay/select 动作以及 dark 主题）。

---

## 3. 功能逻辑：一次 `setOption` 到底发生了什么

这是理解整个项目的钥匙。ECharts 用一段**结构化注释**把更新周期定义为 `EC_CYCLE`（`src/core/echarts.ts:211-277`），实现严格遵循该契约。

### 3.1 周期分类（源码即规范）

| 周期 | 触发者 | 包含步骤 |
|---|---|---|
| `EC_FULL_UPDATE_CYCLE` | `setOption` / `dispatchAction` / `resize` / lazyUpdate 的下一帧 | `EC_PREPARE`（可选）→ 坐标系 create → 数据加工（可选）→ 坐标系 update（可选）→ 视觉编码（可选）→ 布局（可选）→ 渲染 |
| `EC_PARTIAL_UPDATE_CYCLE` | `dispatchAction` | 省略 prepare 与坐标系 create/update，只重跑部分阶段，属性能捷径，需谨慎保证一致性 |
| `EC_PROGRESSIVE_CYCLE` | 由上面两者触发 | 每帧只处理一小批数据，直到 `scheduler.unfinished` 为 false |
| `EC_APPEND_DATA_CYCLE` | `appendData` | 仅追加数据，不改坐标系 extent |
| `SERIES_SPECIFIC_CYCLE` | 如图 `graph` 力导向 | 自行在每帧做布局+渲染 |

两条硬性规则（`229-277`）：

1. **不允许嵌套**：周期进行中再次进入会被延迟（`dispatchAction`）或报错（其他 API）。实现上是 `IN_EC_CYCLE_KEY = '__flagInMainProcess'`（`280`）这个私有标记 + `_pendingActions` 队列（`498`）。
2. **用户可见事件全部在周期之外触发**（`flushPendingActions` / `triggerUpdatedEvent`，`2274-2284`），避免事件回调里改状态导致重入。
3. **模型只能在周期起点修改**：`prepare()`（setOption）、action handler（dispatchAction）、`appendData`。
4. **坐标系/Axis/Scale 实例每个 FULL_UPDATE 重建一次**（`Axis` 类注释 `src/coord/Axis.ts:56` 亦强调 "Lifetime: recreate for each main process"），这让大量缓存可以直接挂在实例上。

### 3.2 `setOption` 主流程（`src/core/echarts.ts:741-820`）

```
setOption(option, {notMerge, lazyUpdate, silent, replaceMerge, transition})
├─ 重入/已销毁检查（742-752）
├─ IN_EC_CYCLE = true; updateECUpdateCycleVersion()      // 版本号用于识别"过期渲染"
├─ 若首次或 notMerge：new OptionManager + new GlobalModel → init()
├─ ecModel.setOption(option, {replaceMerge}, optionPreprocessorFuncs)
│    ├─ OptionManager.setOption → clone + parseRawOption + doPreprocess（每个子 option 跑 preprocessor）
│    └─ GlobalModel._resetOption → mountOption + initBase / _mergeOption
│         └─ topologicalTravel(新增 mainType, 全部 mainType)  // 按 static dependencies 拓扑排序
├─ lazyUpdate ? 存为 PENDING_UPDATE + zr.wakeUp() : prepare() + update()
├─ 非 SSR 时同步 this._zr.flush()    // 保证 setOption 返回后能取像素
└─ IN_EC_CYCLE = false → flushPendingActions → trigger('updated')
```

### 3.3 `prepare` 与"视图复用"（`1670-1770`）

```js
prepare = () => {
    resetCachePerECPrepare(model);          // util/cycleCache.ts
    scheduler.restorePipelines(zr, model);  // 为每个 series 重建 pipeline（progressive 阈值/step）
    scheduler.prepareStageTasks();          // 为每个 stage handler 建 seriesTask / overallTask + stub
    prepareView(ecIns, true);               // 组件视图
    prepareView(ecIns, false);              // 图表视图（同时 prepareView 把 renderTask 接进 pipeline）
    scheduler.plan();                       // 从 tail 向上找 block 任务，确定 blockIndex
}
```

`prepareView` 里的**视图复用键**是 `'_ec_' + model.id + '_' + model.type`（`1716`）。同一 key 复用旧 view 以支持过渡动画；`replaceMerge` 场景会通过 `__requireNewView`（`1712-1714`、`474`）强制新建。未被标记 `__alive` 的 view 会被 dispose 并从 zr 中移除（`1754-1769`）。

### 3.4 `update`：真正的全量流水线（`1880-1938`）

```js
update(payload, updateParams) {
    resetCachePerECFullUpdate(ecModel);
    ecModel.setUpdatePayload(payload);
    scheduler.restoreData(ecModel, payload);          // 数据回滚 + 所有 overallTask dirty
    scheduler.performSeriesTasks(ecModel);            // 每个 series 的 dataTask.perform()（数据初始化/恢复）
    coordSysMgr.create(ecModel, api);                 // **重建**坐标系
    lifecycle.trigger('coordsys:aftercreate', ...);
    scheduler.performDataProcessorTasks(ecModel, payload);  // block 模式：dataZoom 过滤、堆叠、采样…
    updateStreamModes(this, ecModel);                 // 决定 progressiveRender / large
    coordSysMgr.update(ecModel, api);                 // 计算轴 extent、布局框
    clearColorPalette(ecModel);
    scheduler.performVisualTasks(ecModel, payload);   // 视觉编码 + 布局
    zr.setBackgroundColor(...); zr.setDarkMode(...);
    render(this, ecModel, api, payload, updateParams);
    lifecycle.trigger('afterupdate', ecModel, api);
}
```

**注意 `updateStreamModes` 的位置**（`1914-1920`）：它在 `coordSysMgr.update` 之前调用，源码注释解释了权衡——这样 `barGrid` 的 extent 修正能拿到 stream mode 信息，代价是拿不到完整坐标信息（而当前 stream mode 不需要）。

### 3.5 `render` 与渲染顺序（`2430-2539`）

```js
render() {
    allocateZlevels(ecModel);            // 处理多 zlevel 冲突，series 永远在 component 之后
    renderComponents();                  // 组件先渲染（背景、轴、图例…）
    chartsViews.forEach(v => v.__alive = false);
    renderSeries();                      // 图表视图渲染
    chartsViews.filter(!__alive).forEach(v => v.remove(...));
}
```

`renderSeries`（`2472-2539`）不只是"调 render"，它按固定顺序串联 5 个生命周期钩子：

```
series:beforeupdate → 每 series: clearStates → renderTask.perform → updateBlend → 更新选中态
→ series:layoutlabels（LabelManager 统一做标签避让）
→ series:transition（universalTransition）
→ 每 series: updateZ → updateStates
→ updateHoverLayerStatus
→ series:afterupdate
```

这就是"**标签布局必须在所有图元就绪之后、状态切换必须在标签布局之后**"这条设计约束的落地点（源码注释 `2530-2531`：`label should be in normal status when layouting`）。

### 3.6 局部更新（partial update）

`dispatchAction` 会读 `ActionInfo.update` 字段（格式 `'mainType.subType:method'` 或 `':method'`，`2154-2156`），据此走不同捷径（`1868-2083`）：

| update 值 | 行为 |
|---|---|
| `update` / 未指定 | 完整 `update()` |
| `updateView` | 只跑 visualTasks + 全量 render |
| `updateVisual` | 清视觉 → 只跑 `visualType:'visual'` 任务 → 组件/图表 `updateVisual` |
| `updateTransform` | 只让支持 `updateTransform` 的 view 改变换（roam/缩放），不重建图形 |
| `updateLayout` | 已标记 `@deprecated`，内部转调 `update`（`2080-2082`） |
| `none` | 不改视图 |

`updateTransform` 中 `progressiveRender` 的 series 会被强制置脏走完整渲染（`1977-1990`），因为 `LargeSymbolDraw/LineDraw` 的 `updateLayout` 不支持渐进——这是"捷径 + 例外"的典型写法。

### 3.7 渐进渲染与 Task 流水线

**Scheduler**（`src/core/Scheduler.ts`）为每个 series 维护一条 pipeline（`restorePipelines:235-257`）：

```ts
pipeline = {
    id: seriesModel.uid,
    head, tail,                 // Task 单链表
    threshold: getProgressiveThreshold(),   // 默认 3000
    progressiveEnabled: canvas && getProgressive() !== false && !preventIncremental(),
    blockIndex: -1, step: round(progressive || 700), count
}
```

`_pipe()`（`544-552`）把 series 的 `dataTask`、各 stage handler 的 task、视图的 `renderTask` 依次串成链表，并记录 `__idxInPipeline`。

**Task**（`src/core/task.ts`）是一个手写的"分片迭代器"：

- `plan()` 返回 `'reset'` 表示需要重跑；
- `_doReset()` 设置 `_dueIndex/_dueEnd`，并**把下游置脏**（`294-295`），保证数据变化向下传播（`EC_TASK_DIRTY` 注释 `245-252`）；
- `perform({step, modBy, modDataCount})` 每次只推进 `step` 条，返回 `unfinished()`；
- `modBy/modDataCount` 实现**分片采样**（`356-391`），让渐进渲染时图元在空间上均匀铺开，而不是从左往右刷——这是"缩放大数据时视觉不跳动"的关键细节。

`Scheduler.getPerformArgs`（`186-206`）决定某个 task 是否走 increment：`progressiveEnabled && pipeline.context.progressiveRender && task.__idxInPipeline > blockIndex`。

**帧驱动**在 `_onframe`（`620-704`）：每帧 `remainTime = 1`ms 预算内循环执行 `performSeriesTasks → performDataProcessorTasks → updateStreamModes → performVisualTasks → renderSeries(…, 'remain')`，直到时间用尽或 `unfinished` 为 false；完成后显式 `zr.flush()` 以便触发 `finished` 事件。

**图表侧配合**：`ChartView.renderTask`（`src/view/Chart.ts:142-147`）在 reset 时按 `pipelineContext.progressiveRender` 选择 `incrementalPrepareRender` 还是 `render`（`275-298`），`progressMethodMap`（`300-320`）把 `render` 包成 `progress` 并带 `forceFirstProgress`，从而让 `appendData`（少于阈值时）也能工作。

`large` 模式是另一条独立路径：`bar`（`BarView.ts:1101` 的 `LargeBar`）、`candlestick`、`scatter/effectScatter`（`LargeSymbolDraw`）、`lines`（`LargeLineDraw`）把成千上万个图元合并成**一个 Path** 的 `points` 数组，再用 `throttle` 的 mousemove 反查 dataIndex（`BarView.ts:1196-1225`）。

### 3.8 `appendData`（`1634-1663`）

只把数据 push 到 `getRawData()`，置 `scheduler.unfinished = true` 并 `wakeUp`。源码明确提示：**不改动坐标系 extent**，用户需显式给定 category 轴数据或数值轴 `min/max`，否则位置会反复漂移。

### 3.9 生命周期钩子枚举

`src/core/lifecycle.ts:55-65` 定义全部钩子：

```ts
'afterinit' | 'coordsys:aftercreate' | 'series:beforeupdate' | 'series:layoutlabels'
| 'series:transition' | 'series:afterupdate' | 'afterupdate'
```

公开注册入口是 `registerUpdateLifecycle` / `registerPostInit` / `registerPostUpdate`（`src/core/echarts.ts:3091-3095`、`3079-3089`）。

---

## 4. 功能逻辑：模型层

### 4.1 三层模型对象

| 类 | 文件 | 职责 |
|---|---|---|
| `Model` | `src/model/Model.ts`（259 行） | 通用 option 访问器：`get/getShallow/getModel` 支持 `'a.b.c'` 与父模型回退（`_doGet:216-242`）；mixin 了 itemStyle/lineStyle/areaStyle/textStyle |
| `ComponentModel` | `src/model/Component.ts` | 组件基类：`mergeDefaultAndTheme`、`getDefaultOption`（兼容 ES class 与 `extend` 两套写法，`252-283`）、`getReferringComponents`（`293+`）、静态 `dependencies`/`layoutMode` |
| `SeriesModel` | `src/model/Series.ts` | 图表基类：`dataTask`、`pipelineContext`、`getInitialData`、`getData/setData`、选择态（`602-664`）、`isAnimationEnabled`（`548-562`） |
| `GlobalModel` | `src/model/Global.ts`（976 行） | 全局模型：`_componentsMap`、`_seriesIndices`、主题/locale 模型、`setOption/_mergeOption/resetOption` |

**特别注意一个反直觉约定**：`ChartView`/`ComponentModel`/`SeriesModel` 的注释都写着"**成员不得在构造器或声明处初始化**"（`src/view/Chart.ts:100-108`、`src/model/Component.ts:55-63`）。原因是 `enableClassExtend` 同时支持 `class B extends A` 与 `A.extend({...})` 两种继承（`src/util/clazz.ts:77-129`），而类字段初始化会在 `super()` 后覆盖 `extend` 挂上的原型属性。这是**历史包袱直接塑造了编码规范**的典型案例。

### 4.2 option 合并语义（`src/model/OptionManager.ts` + `Global.ts`）

```
rawOption
  ├─ clone（防用户复用对象，105）
  ├─ parseRawOption（294-378）：拆成 baseOption / timelineOptions / mediaDefault / mediaList
  │    └─ 对每个子 option 依次跑 optionPreprocessorFuncs（backwardCompat、aria、axisPointer、graphic、timeline、parallel…）
  └─ GlobalModel.setOption
       ├─ 首次：initBase(option)
       └─ 再次：_mergeOption(newOption, opt)（312-512）
            ├─ 无对应 ComponentModel 类的 mainType（color/animation/textStyle…）→ 直接深合并
            ├─ 有类的 mainType → topologicalTravel 拓扑排序后 visitComponent
            │    ├─ mappingToExists(oldList, newList, mergeMode)
            │    │    mergeMode = !oldList ? 'replaceAll' : replaceMerge ? 'replaceMerge' : 'normalMerge'
            │    ├─ 命中同构类 → mergeOption + optionUpdated(newOpt, false)
            │    └─ 未命中 → new Class(...) + init + optionUpdated(null, true)
            └─ series 处理完 → reCreateSeriesIndices（过滤态索引映射）
```

`replaceMerge` 会留下"洞"（`_componentsMap` 中 `undefined`），所以额外维护 `_componentsCount`（`Global.ts:172`）。`getOption()`（`517-546`）返回时会把内部组件（`isComponentIdInternal`）剔除并按历史兼容规则裁剪数组尾部。

**Media Query 与 Timeline**：`applyMediaQuery`（`385-410`）只支持 `width/height/aspectRatio` 的 `min/max/eq`；`getMediaOption` 用**后定义优先**合并多个命中项（`203`）；`getTimelineOption` 从 timeline 模型取当前帧（`171-188`）。二者都在 `_resetOption` 中被 merge 回主 option。

设计上有一个已知妥协，注释里写得很坦白（`117-147`）：第二次 `setOption` 时"还原 baseOption 再合并"的方案**不是 100% 正确**（组件映射无法与 ecModel 完全一致），所以 `mountOption(isRecreate)` 在非 recreate 场景使用 `_newBaseOption` 而不是回放历史。

### 4.3 开发者体验细节：缺组件检测

`Global.ts:142-154` 的 `checkMissingComponents` 与 `421-438` 的分支，会在 `__DEV__` 下把"用了但没 import"的情况翻译成人话：

```
Component grid is used but not imported.
import { GridComponent } from 'echarts/components';
echarts.use([GridComponent]);
```

这套映射表就是 `BUITIN_COMPONENTS_MAP` / `BUILTIN_CHARTS_MAP`（`83-138`），需要与 `src/export/*.ts` 手工保持同步——属于**必须人工维护的重复知识**。

---

## 5. 功能逻辑：数据层

数据层是 v5 之后重构最彻底的部分，核心思想是**把"原始数据"与"系列数据视图"解耦**。

```
dataset.source / series.data
        │  Source（识别 5 种格式，Source.ts:49-83）
        ▼
   SourceManager（每个 series 一个，处理 fromDatasetIndex/Id + transform 管道）
        │  SeriesDataSchema（维度裁剪：只保留坐标轴/encode 用到的维度）
        ▼
   SeriesData（系列数据视图：维度名↔索引、视觉、布局、diff）
        │
   DataStore（列式存储：Float64Array / Int32Array / Array + 索引数组）
```

### 5.1 格式识别与维度推导

`Source` 支持 `original / arrayRows / objectRows / keyedColumns / typedArray`（`Source.ts:49-83`）。`seriesLayoutBy` 决定行列方向；`sourceHeader` 决定从第几行开始是数据。

维度来源有三层优先级：

1. 坐标系声明的维度（`getDimensionsInfo` / `dimensions`，见 `src/coord/CoordinateSystem.ts:41-47`）；
2. `series.encode` 用户显式映射；
3. `encodeDefaulter` 默认推导（仅在有 dataset 时生成，见 `src/chart/helper/createSeriesData.ts`）。

**维度裁剪**是性能关键：`SeriesDataSchema`（`src/data/helper/SeriesDataSchema.ts:38-48` 注释）说明——当 dataset 维度极多（>30）时，只把"被使用"的维度放进 `dimensionDefineList`，其余通过 `getSourceDimensionIndex` 按名字回查。

### 5.2 存储与索引

`DataStore` 为每个维度分配一个 typed array（`DataStore.ts:50-57`）：

```ts
'float' → Float64Array, 'int' → Int32Array, 'ordinal' → Array, 'number' → Array, 'time' → Float64Array
```

索引数组按数据量自动选型：`rawCount > 65535 ? Uint32Array : Uint16Array`（`119-122`）。所有过滤（`filter`/`selectRange`，`606-661`/`667+`）都**只重排索引、不复制值**，这是 dataZoom 能对 500 万点做实时过滤的原因（源码注释 `664-666` 直言 "Manually inline code, support 5 million data filtering in data zoom"）。

`DataDiffer`（`src/data/DataDiffer.ts`）按 key 计算 add/update/remove，是动画与元素复用的基础。

### 5.3 数据加工流水线

两类 stage handler：

| 类型 | 注册 API | 执行方式 | 典型实现 |
|---|---|---|---|
| 数据加工 | `registerProcessor` | **block 模式**（一次跑完，不渐进，`Scheduler.performDataProcessorTasks:292-295`） | `dataStack`(900)、`legendFilter`(800)、`dataZoomProcessor`(1000)、`axisStatistics`(920)、`dataSample`(5000) |
| 视觉/布局 | `registerVisual` / `registerLayout` | 可渐进（`step` 分片） | `seriesStyleTask`、`symbolTask`、`decal`、`aria`、`barGrid`、`pieLayout`、`forceLayout`… |

优先级常量集中在 `src/core/echarts.ts:158-209`，并作为 `PRIORITY` 对象暴露给扩展作者（`189-209`）。这个数字表是**全框架最需要遵守的隐式契约**——放错位置会导致 extent 计算、过滤顺序、视觉覆盖关系全错（源码注释 `159-171` 明确解释了为什么 dataStack 必须在其他 processor 之前、axisStatistics 必须在 filter 之前）。

### 5.4 数据变换（transform）

`registerTransform` 暴露给用户（`echarts.ts:3341`），内置 `filter`/`sort`（`src/component/transform/install.ts:20-27`）。外部 transform 通过 `ExternalSource`（`src/data/helper/transform.ts:92-200`）拿到只读视图，返回 `{data, dimensions?}`；返回 `null` 的 dimensions 表示"继承上游维度"（`66-81`）。这是 ecStat 等生态包的接入点。

---

## 6. 功能逻辑：坐标系与坐标轴

### 6.1 两级坐标系模型

`src/coord/CoordinateSystem.ts` 定义了清晰的两级抽象（`49-53` 注释）：

- **`CoordinateSystemMaster`**：如 `grid`、`polar`、`geo`、`matrix`。它是"宿主组件"，有 `create/update`、`convertToPixel/convertFromPixel/convertToLayout`、`getAxes`、`getRect`。
- **`CoordinateSystem`**：真正的"执行者"，如 `cartesian2d`、`polar`、`geo`、`single`、`radar`、`parallel`、`calendar`、`matrix`。核心契约是 `dataToPoint` / `pointToData` / `containPoint`，**规定非法值返回 `[NaN, NaN]` 而不是 null**（`140-152`）。

一个 master 可产出多个 executive（grid → 多个 cartesian2d）。`seriesModel.coordinateSystem` 由注入机制填充。

### 6.2 v6 的 `coordinateSystemUsage`（重点新设计）

`src/core/CoordinateSystem.ts:167-233` 把坐标系用途分为三种：

| 常量 | 语义 | 例子 |
|---|---|---|
| `DATA` | 每个数据项按坐标系定位 | bar 在 cartesian2d |
| `BOX` | 只借用坐标系算"整体矩形/锚点" | pie 中心由 calendar/matrix 决定 |
| `NONE` | 不使用 | — |

默认规则：**series 用 `data`，非 series 组件用 `box`**（`202-204`）。这一设计使"把饼图放进矩阵单元格""把 grid 嵌进 matrix"成为可能，而不需要每个图表单独写适配——`pie/funnel/tree/treemap/sankey` 在各自 `defaultOption` 里显式声明 `coordinateSystemUsage: 'box'`（如 `src/chart/pie/PieSeries.ts:256`）。

`injectCoordSysByOption`（`290-347`）统一负责解析 + 注入 + 错误提示；`decideCoordSysUsageKind`（`186-233`）在 `__DEV__` 下对"非 series 组件声明 data 用法""series 用 box 但坐标系不支持"给出精确报错。这是**把隐式规则显式化**的优良设计。

源码注释（`235-289`）还给出了完整决策矩阵，例如：

- `series.bar` 默认 → DATA=cartesian2d，BOX=none；
- `grid: {coordinateSystem: 'matrix'}` → DATA=none，BOX=matrix；
- `series.graph` + matrix 默认 → DATA=matrix；加 `coordinateSystemUsage:'box'` → DATA=内部 view，BOX=matrix（同时用两种）。

### 6.3 Axis 与 Scale

`Axis`（`src/coord/Axis.ts`）持有 `scale` 与像素 extent，提供 `dataToCoord/coordToData/contain/containData/getTicks`。`Scale`（`src/scale/Scale.ts`）是抽象类 + 类管理器（`enableClassManagement:128`），实现有 `Interval/Log/Ordinal/Time`。

`rawExtentInfo`（`src/coord/scaleRawExtentInfo.ts`，34 KB）是"数据 extent → 轴 extent"的决策器，处理 `min/max/scale/boundaryGap/nice/stack/startValue` 的组合语义；v6 新增 `break`（断轴）体系（`src/scale/break.ts`、`breakImpl.ts`、`src/component/axis/axisBreakHelper*.ts`），用 `ScaleMapper` 把"带断点的数据空间"映射到"连续像素空间"（`src/scale/scaleMapper.ts`）。

### 6.4 漫游（roam）与 View 变换

`src/coord/View.ts`（918 行）是地理/图形类坐标系的公共基类，头部注释（`54-130`）定义了四层变换与一条重要约束：

> `VIEW_COORD_SYS_ANIMATION_PROPS_CONSTRAINT`：动画只允许插值 `x/y/scaleX/scaleY`，以保证坐标系与内容严格对齐。

实现路径是 `dataRect →(rawTrans)→ viewRect →(roamTrans)→ 像素空间`，`center/zoom` 是唯一事实源，交互通过 `dispatchAction({type:'xxxRoam'})` 回写，动画被中断时要同步回 `center/zoom`（`74-89`）。这套"单向数据流"避免了 view 与 model 的双向同步地狱。

---

## 7. 功能逻辑：视觉编码与布局

`src/visual/style.ts` 是最能体现"视觉编码分层"的文件：

| 任务 | 优先级 | 作用 |
|---|---|---|
| `seriesStyleTask` | `VISUAL.GLOBAL` (2000) | 按 `visualStyleAccessPath`（默认 `itemStyle`）取样式，**从调色板补色**，支持 `color` 回调（`67-131`） |
| `dataStyleTask` | `CHART_DATA_CUSTOM` (4500) | 逐项合并 `data[i].itemStyle`，覆盖系列级（`134-171`） |
| `dataColorPaletteTask` | `CHART_DATA_CUSTOM` (4500) | `colorBy: 'data'` 时按数据项分配调色板，按 `type + colorBy` 分组 scope（`175-231`） |
| `seriesSymbolTask` / `dataSymbolTask` | GLOBAL / CHART_DATA_CUSTOM | symbol 与 symbolSize |
| `decalVisualStageHandler` | `VISUAL.DECAL` (7000) | 无障碍纹理，最后覆盖 |

`visualMap` 走 `VISUAL.COMPONENT` (4000)，`brush` 走 `VISUAL.BRUSH` (5000)，`aria` 走 `VISUAL.ARIA` (6000)。**优先级即覆盖顺序**：数据项自定义 (4500) > 组件视觉 (4000)，所以"数据项 itemStyle 能覆盖 visualMap"这条用户可感知的语义，就是靠数字大小实现的。

布局不是 view 的职责：`registerLayout` 的 handler 把结果写进 `SeriesData` 的 `setItemLayout` / `setLayout`，view 渲染时读取。例如 `pieLayout`（`src/chart/pie/pieLayout.ts:33-235`）做角度分配、`minAngle` 修正、`roseType`，并把 `viewRect/r` 写回 `data.setLayout` 供标签布局使用；`barGrid`（`src/layout/barGrid.ts`，24 KB）负责跨系列柱宽分配、堆叠，极坐标版本在 `barPolar.ts`。

---

## 8. 功能逻辑：交互、动作与状态

### 8.1 Action 机制

`registerAction`（`src/core/echarts.ts:3114-3188`）把一个"动作"声明为：

```ts
{ type, event, update, action, refineEvent, publishNonRefinedEvent }
```

- `update` 字段决定局部更新粒度（见 3.6）；
- `refineEvent` 是 v5.5+ 引入的"事件精炼"：动作内部先产出中间事件用于 `connect` 传播，再产出对用户更友好的 refined 事件（`2249-2271`）。`select/unselect/toggleSelect` 都共享 `selectchanged` 事件，正是靠这套机制避免事件名冲突（`3387-3412` 与 `3184-3187` 的冲突检查）。

`dispatchAction`（`1574-1624` + `doDispatchAction 2148-2272`）负责：批量（`payload.batch`）、重入排队、`escapeConnect` 标记、事件发射。`messageCenter` 与实例事件是两套 Eventful（`350-353`），前者用于 connect 广播。

### 8.2 多图表联动 `connect`

`enableConnect`（`2806-2841`）为每个实例在 `messageCenter` 上挂所有已知事件，命中同 group 时用 `makeActionFromEvent` 反查动作类型并转发，同时用 `CONNECT_STATUS_PENDING/UPDATING/UPDATED` 三态**防止环路风暴**（`2831-2837`）。这是一个小而完整的"发布-订阅 + 防重入"实现。

### 8.3 状态系统（emphasis / blur / select）

`src/util/states.ts`（32 KB）：

- 只在元素上**打标记**（`singleEnterEmphasis:118-120`：`Only mark the flag. States will be applied in echarts.ts in next frame`），真正的样式切换在 `applyChangedStates`（`echarts.ts:2551+`）里按帧批量执行——避免高频 hover 触发重排；
- `Z2_EMPHASIS_LIFT = 10`、`Z2_SELECT_LIFT = 9`（`85-86`）用 z2 抬升保证高亮不被遮挡；
- `hoverLayerThreshold`（`globalDefault.ts:124` = 3000）在元素过多时把所有 emphasis 收敛到**单一 hover 层**（`echarts.ts:2592-2633`），并区分 `HOVER_LAYER_FROM_THRESHOLD` / `HOVER_LAYER_FOR_INCREMENTAL`。

### 8.4 事件查询

`ECEventProcessor`（`src/util/ECEventProcessor.ts`）实现 zrender 的 `EventProcessor` 接口，把 `chart.on('click', 'series.line', fn)` 的字符串/对象 query 归一为 `{cptQuery, dataQuery, otherQuery}` 并在 `filter` 中逐字段比对（`107-145`）。`findEventDispatcher`（`src/util/event.ts`）负责从 `e.target` 向上找到"可作为分发器"的元素。

---

## 9. 扩展机制（本项目最重要的架构面）

### 9.1 三处入口的一致性

新增一个图表类型要动三处（这是仓库 AGENTS.md 特别强调的点）：

| 层 | 文件 | 作用 |
|---|---|---|
| 实现 | `src/chart/xxx/{XxxSeries,XxxView,install}.ts` | `install(registers)` 中注册 |
| 模块导出 | `src/export/charts.ts` | `export {install as XxxChart}` |
| 打包入口 | `src/echarts.all.ts` / `common.ts` / `simple.ts` | `use([...])` |

顶层 `src/chart/bar.ts` 这类文件只是 `use(install)` 的一行副作用入口，供 `echarts/lib/chart/bar` 这类旧路径使用。

### 9.2 注册器设计（`src/extension.ts`）

`use()`（`101-123`）做了三件事：数组展开、**按引用去重**（`112-115`）、支持函数或 `{install}` 对象。注册器对象（`49-92`）暴露的能力：

```
registerPreprocessor / registerProcessor / registerPostInit / registerPostUpdate
registerUpdateLifecycle / registerAction / registerCoordinateSystem
registerLayout / registerVisual / registerTransform / registerLoading / registerMap
registerImpl / registerComponentModel / registerComponentView
registerSeriesModel / registerChartView / registerCustomSeries
registerSubTypeDefaulter / registerPainter
+ PRIORITY + 四个基类
```

设计要点：

- **类管理器**（`src/util/clazz.ts:227-351`）用 `'series.line'` 这样的 `mainType.subType` 字符串注册/查找类，`storage` 是模块级闭包。好处是跨包、跨 realm 都能用；代价是**全局单例**，无法为不同实例隔离注册表。
- **`registerImpl`**（`src/core/impl.ts`）把 `registerMap/getMap` 这类"依赖具体组件实现但 API 在 core"的能力延迟注入，避免 core 打包进 geo 代码。这是很克制的**依赖倒置**用法。
- **`registerSubTypeDefaulter`**：让 `legend` / `dataZoom` / `visualMap` / `timeline` 等支持省略 `type` 时给默认值。
- **`registerCustomSeries`**：开放 `renderItem` 生命周期（`CustomView.ts` 1353 行，是全仓第三大文件），支持任意 zrender 图形、过渡、关键帧动画、shape morphing。
- **`registerPainter`**：直接透传 zrender painter，让第三方渲染后端可插拔。

### 9.3 内置扩展点清单（按"注册了什么"看）

以典型安装器为例：

| 安装器 | 注册内容 |
|---|---|
| `chart/bar/install.ts` | ChartView、SeriesModel、2 个 layout（跨系列 + 渐进）、processor(dataSample, STATISTIC)、action(changeAxisOrder)、barGrid 轴 handler |
| `chart/line/install.ts` | ChartView、SeriesModel、layout(points)、visual(图例线样式)、processor(dataSample) |
| `chart/pie/install.ts` | ChartView、SeriesModel、legacy dataSelect action、layout(pieLayout)、processor(dataFilter)、processor(negativeDataFilter) |
| `chart/graph/install.ts` | ChartView、SeriesModel、processor(categoryFilter)、2 个 visual、3 个 layout(simple/circular/force)、coordinateSystem('graphView')、2 个 legacy action、roam action |
| `component/grid/installSimple.ts` | GridView/Model、coordinateSystem('cartesian2d')、x/y 轴模型与视图、preprocessor |
| `component/polar/install.ts` | AxisPointer 类注册、coordinateSystem('polar')、PolarModel/View、angle/radius 轴、layout(barPolar) |
| `component/tooltip/install.ts` | 先 `use(installAxisPointer)`、TooltipModel/View、action(showTip/hideTip) |
| `component/axisPointer/install.ts` | CartesianAxisPointer 类、Model/View、preprocessor、STATISTIC processor 生成 `coordSysAxesInfo`、broadcast action(`:updateAxisPointer`) |
| `component/legend/install.ts` | 聚合 plain + scroll；plain 里注册 Model/View、SERIES_FILTER processor、subTypeDefaulter、action |
| `component/dataZoom/install.ts` | 聚合 inside + slider；common 里注册 FILTER processor、action、subTypeDefaulter |
| `component/aria/install.ts` | preprocessor + `VISUAL.ARIA` visual |
| `component/transform/install.ts` | `registerTransform(filter)` + `registerTransform(sort)` |

### 9.4 扩展点统计（衡量"可插拔程度"的客观指标）

| 扩展点 | 源码调用点 |
|---|---|
| `registerAction` | 54 |
| `registerLayout` | 28 |
| `registerVisual` | 23 |
| `registerProcessor` | 17 |
| `registerCoordinateSystem` | 10 |

结论：**这是一套以"注册表 + 阶段任务"为核心的框架，而不是以继承为核心的类库**。图表/组件的差异被表达为"往同一流水线的不同优先级插任务"，这带来极强的组合性（图 + 组件 + 坐标系可任意搭配），也带来隐式契约（优先级、维度名、dirty 传播）。

---

## 10. 技术架构

### 10.1 整体架构图

```
┌────────────────────────────── 用户 API ──────────────────────────────┐
│ echarts.init / setOption / dispatchAction / resize / getDataURL       │
│ on('click'|'rendered'|'finished') / connect / registerXXX / use       │
└────────────────────────────────┬─────────────────────────────────────┘
                                 │
┌────────────────────────────────▼─────────────────────────────────────┐
│                        ECharts 实例（core/echarts.ts）                │
│  _model: GlobalModel   _scheduler: Scheduler   _api: ExtensionAPI    │
│  _zr: zrender          _chartsViews/_componentsViews                 │
│  EC_CYCLE 状态机：IN_EC_CYCLE / PENDING_UPDATE / CYCLE_VERSION        │
└───┬─────────────┬──────────────┬──────────────┬───────────────┬──────┘
    │             │              │              │               │
┌───▼────┐  ┌─────▼──────┐  ┌────▼─────┐  ┌─────▼──────┐  ┌─────▼─────┐
│ 模型层 │  │ 调度/任务  │  │ 坐标系   │  │ 数据/视觉  │  │ 视图层    │
│Global  │  │Scheduler   │  │CoordSys  │  │SeriesData  │  │ChartView  │
│Series  │  │Task 链表   │  │Mgr+Axis  │  │DataStore   │  │Component  │
│Component│ │pipeline    │  │Scale     │  │VisualMap   │  │View       │
│OptionMgr│ │progressive │  │View(roam)│  │Layout      │  │renderTask │
└───┬────┘  └─────┬──────┘  └────┬─────┘  └─────┬──────┘  └─────┬─────┘
    │             │              │              │               │
┌───▼─────────────▼──────────────▼──────────────▼───────────────▼─────┐
│                          公共基础设施                                │
│ util/{types,model,graphic,number,layout,clazz,states,symbol,format} │
│ label/LabelManager   animation/*   i18n   theme                      │
└──────────────────────────────────┬──────────────────────────────────┘
                                   │
                        ┌──────────▼──────────┐
                        │ zrender 6.1.0 内核  │
                        │ Element 树/动画/事件 │
                        │ Canvas & SVG Painter │
                        └─────────────────────┘
```

### 10.2 关键设计模式

| 模式 | 落点 | 说明 |
|---|---|---|
| **注册表 / 插件** | `extension.ts`、`clazz.ts` 类管理器、`impl.ts` | 全部内置能力都是"插件"，`echarts.use` 是唯一装配入口 |
| **管道-过滤器（分阶段）** | `Scheduler` + `StageHandler` | processor/visual/layout 按 priority 串成有序阶段，每个阶段可 per-series 或 overall |
| **责任链（Task 单链表）** | `task.ts` 的 `_upstream/_downstream` | 数据流严格单向：dirty 向下传播，outputData 向上游拉取 |
| **中介者** | `ExtensionAPI`（`core/ExtensionAPI.ts`） | 向视图层暴露受限 API（`availableMethods:32-51` 白名单），内部方法（`getModel`/`getViewOfSeriesModel`）未暴露给外部 |
| **模板方法** | `ChartView.render` + `incrementalPrepareRender/incrementalRender` + `ComponentView.render` | 基类定骨架，子类填内容 |
| **策略** | `Scale` 子类、`CoordSysUsageKind`、`RoamController`、`dataSample` 的 Sampler 表 | 可替换算法 |
| **享元 / 缓存** | `sharedModel`（style.ts:133）、`cycleCache.ts`、`makeInner` | 高频路径避免建对象 |
| **空对象** | `Model` 的默认 option、`noop` | 减少判空 |
| **闭包式私有** | `private static internalField`（echarts.ts:1668-2842） | 在 `private` 关键字流行前，用 IIFE 把 20+ 个内部函数彻底隐藏 |

### 10.3 关键数据结构选型

| 结构 | 选型 | 理由 |
|---|---|---|
| 数据列 | `Float64Array/Int32Array/Uint16Array/Uint32Array`（`DataStore.ts:50-57,119-122`） | 内存紧凑、GC 友好、数值遍历快 |
| 维度名映射 | `HashMap`（zrender 实现） | 避免原型链污染，`__proto__` 等名称安全 |
| 排序 | `timsort`（`echarts.ts:41,597`） | 对近似有序数据（时间序列）接近 O(n) |
| 最近点查询 | `KDTree`（`src/util/KDTree.ts`） | scatter 大数据 tooltip 命中 |
| 元素遍历 | `zr.storage.traverse` + `eachRendered` | 只遍历本次渲染的图元，渐进下不扫描全量 |
| 每实例私有状态 | `makeInner`（`util/model.ts`） | 不污染用户可见对象，避免属性名冲突 |
| 每帧状态缓存 | 模块级 `inner` + `cycleCache` | 用 EC_CYCLE 版本号失效 |

### 10.4 性能设计清单

| 手段 | 实现位置 | 触发条件 |
|---|---|---|
| 渐进渲染 | `Scheduler` pipeline + `Task.perform(step)` | `canvas && progressive !== false && data.count() > progressiveThreshold(3000)` |
| 分片采样 | `task.ts:356-391` modBy | 渐进时均匀铺开图元 |
| large 模式 | `BarView/LargeBar`、`CandlestickView`、`LargeSymbolDraw`、`LargeLineDraw` | series `large: true`，阈值 400/600/2000 |
| 降采样 | `processor/dataSample.ts`（average/sum/min/max/nearest/lttb/minmax） | `data.count() > 10 && cartesian2d && sampling` |
| hover 层 | `echarts.ts:2592-2633` | 元素数 > `hoverLayerThreshold(3000)` |
| 状态批量应用 | `applyChangedStates` 按帧 | hover 时只打标记 |
| 脏矩形 | `init` 的 `useDirtyRect` | 默认 `false`（`echarts.ts:537`） |
| 事件节流 | `_throttledZrFlush = throttle(flush, 17)`（`583`） | 微信等 rAF 异常环境 |
| 索引不复制 | `DataStore.filter/selectRange` | dataZoom 实时过滤 |
| 维度裁剪 | `SeriesDataSchema` | dataset 维度数过多 |
| 标签避让 | `LabelManager` + `labelLayout` | `labelLayout.hideOverlap` |

`model/globalDefault.ts` 把这些阈值集中成可全局覆盖的默认值（`animationThreshold: 2000`、`progressiveThreshold: 3000`、`progressive: 400`、`hoverLayerThreshold: 3000`）。

### 10.5 类型系统设计

- **`ComposeOption`**（`src/export/core.ts:130-134`）：按需组合 option 类型，是"按需引入"在类型层的对应物。实现上靠 `ExtractComponentOption` + 手工 `Dependencies` 表（`80-122`，把 `grid` 自动带上 `xAxis/yAxis/axisPointer`），并在注释里承认"更严格的注入类型会让错误信息不可读，故暂不启用"。
- **`__DEV__` 常量**（`src/global.d.ts:20`）：`declare const __DEV__: boolean`，编译期被替换成 `process.env.NODE_ENV !== 'production'`（`build/transform-dev.js:40-50`），构建输出再 `replace` 成定值（`build/config.js:46-53`），配合 terser 完成开发日志的 DCE。
- **类型产物多轨**：`types/dist/all.d.ts`（ESM）、`echarts.d.cts`（CJS/UMD）、`echarts.d.ts`（自包含，供在线编辑器）、根 `index.d.ts`（deprecated shim）。成因写得很清楚（`package.README.md:101-115`）：TS 5.6 起 `export =` 与 ESM 不再兼容（TS1203），`NodeNext` 又要求显式扩展名（TS2834/TS2846），且重复声明会触发 TS2442。

### 10.6 发布与打包架构

| 产物 | 生成者 | 格式 | 用途 |
|---|---|---|---|
| `dist/echarts.js` / `.min.js` | `build/build.js --type all --min`（rollup, UMD） | UMD | `main`、CDN、`<script>` |
| `dist/echarts.common/simple` | 同上，不同入口 | UMD | 体积裁剪版 |
| `dist/echarts.esm.js/.mjs` | `--format esm` | ESM | 需 `.mjs` 兼容 `type:module`（`build.js:117-129`） |
| `lib/` + 根 `index*.js` + `core.js` 等 | `build/pre-publish.js`（tsc → ES2015） | ESM 文件树 | `module` 字段、按需引入 |
| `types/` | 同上 `declaration: true` + `rollup-plugin-dts` | `.d.ts` | 类型 |
| `i18n/*.js` / `*-obj.js` | `build/build-i18n.js`（文本包裹 UMD） | UMD | 自注册 / 纯对象 |
| `extension/{bmap,dataTool}`、`ssr/client/*` | pre-publish + `build.js` | ESM + UMD | 独立扩展 |
| `theme/*.js` | 手工维护 | UMD | `registerTheme` |

**tree-shaking 的设计**：`sideEffects` 白名单（`package.json:31-42`）只保留真正的副作用文件（`index*.js`、`lib/echarts.js`、图表/组件安装器、主题、i18n），其余全部可摇。`src/export/charts.ts:21-26` 特意说明为什么用 `export {install as LineChart}` 而非 `export * as`——后者会让 webpack 的 treeshaking 失效。

---

## 11. 图表与组件能力矩阵

### 11.1 图表（23 类）

| 家族 | 图表 | 坐标系 | 复杂度（源码行） |
|---|---|---|---|
| 直角坐标 | `bar`、`pictorialBar` | cartesian2d / polar | 高（2,563 / large + progressive + realtimeSort） |
| 直角坐标 | `line` | cartesian2d / polar | 高（2,214 / symbol/area/step/平滑/渐进） |
| 直角坐标 | `scatter`、`effectScatter` | 6 种坐标系 | 中低（374 / 229） |
| 直角坐标 | `candlestick` | cartesian2d | 中（large 三 Path 复用） |
| 直角坐标 | `heatmap` | cartesian2d / geo / calendar / matrix | 中（667） |
| 饼环 | `pie` | box（默认）/ 任意 box 坐标系 | 高（标签布局极复杂） |
| 饼环 | `sunburst`、`funnel`、`gauge` | 自带径向/盒布局 | 中 |
| 关系层级 | `graph` | view + 可挂任意 | 高（2,040 / 3 种布局 + 力导漫游） |
| 关系层级 | `tree` | box + view | 高（1,504 / 展开折叠） |
| 关系层级 | `treemap` | box | 最高（2,639 / 布局 + 面包屑） |
| 关系层级 | `sankey`、`chord` | box / none | 中高 |
| 地理 | `map`、`lines` | geo | 中 |
| 专业坐标 | `radar`、`parallel`、`themeRiver`、`boxplot` | radar / parallel / singleAxis / cartesian2d | 中低 |
| 自定义 | `custom` | 任意或 none | 很高（`CustomView` 1,353 行） |

### 11.2 组件（23 个功能目录）

| 类别 | 组件 |
|---|---|
| 坐标系宿主 | `grid`（+`gridSimple`）、`polar`、`geo`、`radar`、`parallel`、`singleAxis`、`calendar`、`matrix` |
| 数据 | `dataset`、`transform`、`dataZoom`（inside/slider/select） |
| 交互 | `tooltip`、`axisPointer`、`brush`、`toolbox`、`timeline`、`thumbnail` |
| 图例标注 | `legend`（+plain/scroll）、`markPoint`、`markLine`、`markArea`、`title`、`graphic` |
| 视觉无障碍 | `visualMap`（continuous/piecewise）、`aria` |
| 基础设施 | `axis`（轴视图与断轴）、`helper`（RoamController/BrushController/MapDraw/sliderMove） |

### 11.3 关键协作链路（最能体现架构的三条）

**① tooltip ↔ axisPointer ↔ grid/axis**

```
mousemove → globalListener(throttle) → dispatchAction({type:'updateAxisPointer'})
  → axisTrigger 计算 dataByCoordSys → dispatchAction({type:'showTip', dataByCoordSys})
  → TooltipView._showAxisTooltip → 各 series.formatTooltip → TooltipHTMLContent/RichContent
```

`axisPointer` 在 `PROCESSOR.STATISTIC` 阶段用 `collect()` 生成 `coordSysAxesInfo`（`component/axisPointer/install.ts:56-63`），`update` 值为 `':updateAxisPointer'`（无 mainType，broadcast 到所有 view）。

**② dataZoom ↔ Axis ↔ data**

```
拖动/滚轮 → dispatchAction({type:'dataZoom'}) → DataZoomModel 记录 start/end
  → FILTER processor → AxisProxy.filterData → scale 重算 → DataStore.selectRange（只换索引）
  → 视觉/布局重跑 → 只有可见数据渲染
```

`AxisProxy`（22 KB）负责多个 dataZoom 共享同一轴时的窗口合成，`history.ts` 支持 toolbox 的还原。

**③ visualMap ↔ SeriesData ↔ 图元**

```
visualMap(continuous/piecewise) → VISUAL.COMPONENT(4000)
  → 匹配 seriesTargets → 取 dimension → VisualMapping 映射 → data.setItemVisual
  → ChartView.render 读 getItemVisual 生成/更新图元样式
```

`seriesTargets` 是 v6 新增的多目标能力（`VisualMapModel.ts:100-105`），允许一个 visualMap 同时作用于多个 series 的不同维度。

---

## 12. 工程化：构建、测试、CI

### 12.1 构建链（两条）

```
开发：esbuild（build/dev-fast.js）
  src/echarts.all.ts --bundle--> dist/echarts.js（cjs + 手写 UMD 外壳，define __DEV__=true，watch）

发布：tsc + rollup（build/pre-publish.js → build/build.js）
  src/**/*.ts --tsc(ES2015, declaration)--> tmp/ + types/
    └─ after: tmp/src → lib/，tmp/src/echarts.all.js → 根 index.js，替换 zrender/src→zrender/lib、补 .js 后缀
  rollup(index*.js) --replace(NODE_ENV) + terser--> dist/*.js
  rollup-plugin-dts --> types/dist/{shared,all,charts,...}.d.ts + echarts.d.cts
```

关键机制：

- **`__DEV__` 三段式裁剪**：TS 声明 → Babel 替换成 `process.env.NODE_ENV !== 'production'` → rollup `replace` 定值 → terser DCE。`build.js:144-153` 的 `checkBundleCode` 本应在产物里复查是否还有 `process.env.NODE_ENV` 残留。
- **入口生成**：`generateEntries()`（`pre-publish.js:514-527`）从 `build/template/*.js` 复制出根 `core.js/charts.js/...`，保证"物理文件存在"，避免依赖 `exports` 别名（成因见 `package.README.md:37-40`）。
- **i18n 生成**：`build-i18n.js:74-108` 把 `src/i18n/langXX.ts` 用正则把 `export default` 换成 `var localeObj =`，再包一层 UMD，分别生成"自注册版"（调 `echarts.registerLocale`）与"纯对象版"（`*-obj.js`）。

### 12.2 测试三层

| 层 | 命令 | 入口 | 规模 | 覆盖对象 |
|---|---|---|---|---|
| 单元 | `npm test` | `test/ut/jest.config.cjs`（ts-jest + jsdom + jest-canvas-mock，`__DEV__: true`） | 26 个 spec | `src`，偏 model/data/scale/util/api |
| 视觉回归 | `npm run test:visual` | `test/runTest/server.js` + Puppeteer | 606 HTML、273 actions、243 marks | `dist/echarts.js`，对比"另一版本运行时"的截图 |
| 类型 | `npm run test:dts` | `build/testDts.js`（TS 3.5.0 起多版本，strict，`typeRoots: []`） | 5 个 fixture | `lib/` + `types/dist`，只开 `NodeNext` 一组配置 |

视觉回归的两个设计亮点：

- **无入库基准图**：expected 来自 npm release/nightly 或 PR 预览，actual 来自当前构建——基准是"另一个版本"，避免图片仓库膨胀，但**双方同错会漏检**。
- **确定性**：seededrandom 替换 `Math.random`、固定 16ms 帧 + mock `Date`/timers、禁用 CSS 动画。
- 判定阈值不一致：失败阈值 `diffRatio > 0.0001`（`test/runTest/store.js:342`），报告只显示 `> 0.001`（`genReport.js:120`）。

### 12.3 CI / 发布

| 工作流 | 触发 | 内容 |
|---|---|---|
| `ci.yml` | PR opened/reopened/synchronize | lint（**仅改动过的 `src/**/*.ts`**）→ `build:lib` → `checktype`；另一 job：`test` → `npm run release` → `test:dts` |
| `nightly.yml` | 每日 09:00 + 手动 | 装 zrender-nightly → release → test → test:dts → `npm publish` |
| `nightly-next.yml` | 每日 09:10 | 同上，`next` 分支、`--tag next` |
| `source-release.yml` | `release: prereleased` | 生成投票材料 → 打源码包 → 解包重建并跑测试 |
| `stale.yml` / `update-notice-year.yml` | 定时 | 社区维护自动化 |

CI 盲区：**不跑视觉回归**、**不跑 `checkheader`**、lint 只覆盖 `src` 下的 TS（`ssr/client/src`、`build/*.js`、`test/**`、`i18n/` 未覆盖）、Node 矩阵仅 20.x。

### 12.4 代码质量约束

| 约束 | 位置 | 内容 |
|---|---|---|
| ESLint 通用 | `.eslintrc-common.yaml` | 单引号、分号、4 空格、stroustrup、max-len 120、`curly: all`、`eqeqeq: allow-null`、禁 `eval`/`with`/原生扩展 |
| ESLint 生产 | `.eslintrc-common-production.yaml` | 禁 spread/默认参数/async，禁 `Promise/Map/Set/Symbol/Proxy/Reflect` 等新 API（保证 ES3 目标可跑） |
| TS 编译 | `tsconfig.json` | `target: ES3`、`noImplicitAny`/`noImplicitThis`/`strictBindCallApply`、`declaration`、`importHelpers`（→ tslib） |
| License header | `build/preamble.js` + `build/headerUtil.js` + `.headerignore` | 按扩展名维护头注释；`checkHeader` 缺失即 exit 1，`addHeader` 自动补 |
| pre-commit | `.husky/pre-commit` | `npm run lint` + `npm run checktype` |

---

## 13. 系统设计评估

### 13.1 优点

1. **声明式 + 编译式渲染的统一**：用户只写 option，框架负责把"意图"编译成图元树。`EC_CYCLE` 注释（`echarts.ts:211-277`）是全仓质量最高的设计文档，把"什么能改模型、什么时候能 dispatch、坐标系何时重建"讲清楚，使十几万行级别的演进仍能保持一致的更新语义。
2. **以"阶段任务 + 优先级"代替继承**：23 类图表、23 个组件共享同一套 processor/visual/layout 管线，组合自由度极高（图表 × 坐标系 × 组件几乎正交）。`PRIORITY` 常量化让扩展作者无需读内核。
3. **性能为第一约束的取舍到处可见**：索引不复制、typed array、分片采样、large 模式、hover 层、状态按帧批量应用、渐进渲染按 1ms 预算切帧。注释里保留了取舍理由（如 `updateStreamModes` 的位置、`hoverLayerThreshold` 应 ≤ `progressiveThreshold`）。
4. **`coordinateSystemUsage` 是漂亮的抽象升级**：把"数据定位"与"盒子定位"区分开，用一个枚举解决了历史上大量 `if (coordSys.type === ...)` 特判，并附带完整决策矩阵注释。
5. **数据层解耦彻底**：`Source → SourceManager → SeriesData → DataStore` 四层职责清晰，transform 机制对外只暴露 `ExternalSource` 只读视图，生态扩展（ecStat）无需触碰内部。
6. **开发者体验有专门投入**：缺组件报错带 import 代码（`Global.ts:142-154`、`421-438`）；`__DEV__` 全链路裁剪让生产包零日志；`getOption` 自动剔除内部组件；`convertToPixel/convertFromPixel` 跨坐标系自动匹配（`echarts.ts:2106-2138`）。
7. **扩展与内核的边界克制**：`ExtensionAPI` 白名单（`ExtensionAPI.ts:32-51`）明确区分"给视图用的 API"与"内部 API"；`registerImpl` 用依赖倒置避免 core 打包 geo。
8. **SSR 方案轻量**：不加第二套渲染器，直接复用 SVG painter + `renderToString`，客户端用 `hydrate` 读 `ecmeta_*` 属性恢复事件（`ssr/client/src/index.ts:42-90`），`PieView`/`LineView`/`LegendView` 中 `ecModel.ssr` 分支只是"禁用动画、写入元数据"。

### 13.2 问题与风险（按严重度排序，均附证据）

**P0 — 类型安全存在系统性缺口**

- `tsconfig.json:3-7` 未开 `strictNullChecks`（也未 `strict: true`），仅开 `noImplicitAny/noImplicitThis/strictBindCallApply`。
- 全仓 `as any` **272 处**、`@ts-ignore/@ts-expect-error` **34 处**、`eslint-disable` **42 处**。
- `src/util/types.ts` 单文件 2,211 行，是典型的"巨型类型文件"，修改时极易产生连锁类型错误。
- 后果：`null` 分支的保护完全依赖人工；`echarts.ts` 中大量 `private` 方法通过 `(this as any)` / `@ts-ignore` 绕过（如 `2848-2862` 的 `one()`）。

**P0 — 核心文件单体化，认知与协作成本高**

| 文件 | 行数 |
|---|---|
| `src/core/echarts.ts` | 3,434 |
| `src/util/types.ts` | 2,211 |
| `src/component/axis/AxisBuilder.ts` | 1,450 |
| `src/chart/custom/CustomView.ts` | 1,353 |
| `src/data/SeriesData.ts` | 1,321 |
| `src/util/model.ts` | 1,314 |
| `src/chart/line/LineView.ts` | 1,293 |
| `src/data/DataStore.ts` | 1,156 |
| `src/component/tooltip/TooltipView.ts` | 1,120 |

`echarts.ts` 一个文件同时承担：实例 API、EC_CYCLE 状态机、prepare/update/render、zlevel 分配、事件绑定、connect、注册表、内置能力注册。任何改动都要在 3,400 行里定位，且它被 30+ 模块 import，是事实上的**编译与理解瓶颈**。

**P1 — 模块级全局单例阻碍多实例隔离**

`src/core/echarts.ts:2891-2914` 的 `optionPreprocessorFuncs / visualFuncs / dataProcessorFuncs / actions / themeStorage / loadingEffects / instances / connectedGroups` 全在模块作用域；`clazz.ts` 的类注册表同理。影响：

- 无法在同一进程内为不同实例配置不同注册集（SSR 多租户、微前端隔离场景受限）。
- `Scheduler` 在构造时对 handler 数组做 `slice()` 快照（`Scheduler.ts:146-148`），源码注释已承认"实例创建后新增 processor 不受支持"。
- 测试之间存在隐式耦合（`registerLayout` 用全局 `registeredTasks` 去重，`normalizeRegister:3268-3271`）。

**P1 — 隐式契约多，错误发现晚**

- 优先级数字表（`158-209`）必须精确匹配语义，放错不报错只是"渲染结果不对"。
- 维度名（`x/y/value/lng/lat/angle/radius` 等）靠字符串约定，拼错只会在运行时出现 NaN 或空图。
- `static dependencies` 仅在 14 个类中声明（`BarSeries`/`LineSeries`/`GridModel`/`PolarModel`/`BrushModel`/`TooltipModel`/`ThumbnailModel` 等），漏写会导致组件解析顺序错误。
- `coordinateSystemUsage` 虽新增了 `__DEV__` 校验，但只在"显式声明且非法"时报错，隐式路径仍宽松。
- 全仓 `TODO/FIXME/PENDING` **647 处，分布于 227 个文件**——说明有大量"已知未完成"的妥协散落各处。

**P1 — 数据层与视图层对"内部结构"的耦合仍存**

`chart/helper/createSeriesData.ts`、`line/LineView.ts` 等处直接操作 `SeriesData` 的 `_store`、`_rawData`、`getStore()`；`visual/style.ts:149-168` 为性能直接用 `sharedModel.option = rawItem[stylePath]` 并注释 "Not use getItemModel for performance consideration"。这些是**为性能牺牲封装**的显式选择，但会让内部重构（如替换存储引擎）代价极高。

**P2 — 构建链的文本处理脆弱**

- `pre-publish.js:506-511` 用 `new Function` 解析 tsconfig（因为有注释）。
- `pre-publish.js:312-379` 用正则改 import 路径、补 `.js` 后缀、替换 `zrender/src → zrender/lib`。
- `build-i18n.js:94-105` 用正则把 `export default` 替换成 `var localeObj =`。
- `build.js:144-153` 的 `checkBundleCode` 调用 `transformDEV.recheckDEV(code)` 却**丢弃了返回值**（`transform-dev.js:56-58` 返回 boolean），因此"产物里残留 `process.env.NODE_ENV`"不会失败；`validateIO(undefined, undefined)`（`build.js:89`）也是死代码。
- 隐式前置依赖：`dist/extension`、`ssr/client/dist` 依赖根 `index.js` / `extension/` / `ssr/client/lib` 已由 `prepare` 生成，脚本只打印提示（`build.js:177-183`）。

**P2 — 注释与实现漂移**

`build/pre-publish.js:20-29` 头注释仍称 "Build all files to **CommonJS** to `echarts/lib`"，实际是 `module: 'ES2015'`（`96-97`）；`generateEntries` 中 `if (entryPath !== 'option')` 恒为真（`514-519`），`build/template/option.d.ts` 成为孤儿。这类漂移会误导新贡献者。

**P2 — 测试策略的盲区**

- 单元测试 `__DEV__` 恒为 `true`（`jest.config.cjs:35-37`），**生产分支（`__DEV__ === false` 的 DCE 路径）无任何覆盖**。
- 视觉回归是"两版本对比"，非"基准图对比"，双方同错漏检；且 CI 不执行。
- `testDts.js` 准备了 6 组 tsconfig 组合，实际只启用 `NodeNext + ESM`（`86-93`，其余被注释），fixture 仅 5 个文件。
- `test/node` 只有 2 个手工脚本，不在 CI。
- 无内存泄漏 / 长时间运行的回归（`dispose` 后引用清空逻辑见 `echarts.ts:1427-1443`，靠人工保证）。

**P2 — 安全相关面需要持续关注**

- tooltip `renderMode: 'html'` 路径使用 `innerHTML`（`TooltipHTMLContent.ts:410/443/453/457/471`）；框架对字符串形式的组件 tooltip content 默认开启 `encodeHTMLContent`（`TooltipView.ts:749-767`），但**用户 formatter 返回的 HTML 不做转义**，属需文档化的风险。
- `appendTo` 支持任意 CSS 选择器（`TooltipHTMLContent.ts:315-321`），截图/挂载行为依赖用户输入。
- 仓库自带 `test/tooltip-xss.html` 用例，说明团队有意识覆盖该风险；`AGENTS.md` 也引用了 Security Checklist。

**P3 — 发布流程的人工耦合**

`dist/`（22 个文件）、`i18n/`（55 个）、`theme/`（45 个）被 git 跟踪，发布需本地构建后人工提交；nightly 用 `NPM_TOKEN` 直接发布且无 provenance 证明。

### 13.3 改进建议

| 优先级 | 建议 | 落点 |
|---|---|---|
| 高 | 渐进开启 `strictNullChecks`（先从 `src/data`、`src/scale`、`src/util/*` 等叶子模块开始，用 `// @ts-nocheck` 隔离存量） | `tsconfig.json` |
| 高 | 拆分 `core/echarts.ts`：至少把"注册表 + 内置注册"抽到 `core/registry.ts`，"EC_CYCLE 状态机"抽到 `core/cycle.ts`，"事件绑定/connect"抽到 `core/interaction.ts` | `src/core/` |
| 高 | 修复 `checkBundleCode` 未使用返回值（改为断言 `!transformDEV.recheckDEV(code)`），并补 `--type` 白名单与前置产物存在性检查 | `build/build.js:144-153` |
| 中 | 把优先级/维度名/`dependencies` 的隐式契约做成注册期可校验断言（如 `registerVisual` 校验 priority 落在允许区间） | `normalizeRegister:3244` |
| 中 | 为注册表引入"实例级 override"（保留全局默认，允许 `init` 传入 registry），解除多实例隔离限制 | `extension.ts` + `ECharts` 构造 |
| 中 | CI 增加：`checkheader`、`ssr/client/src` 与 `build/*.js` lint、至少一次 `__DEV__=false` 的单测跑批、视觉回归子集 smoke | `.github/workflows/ci.yml` |
| 中 | 视觉回归引入"入库基准图 + 定期刷新"，或至少对 axis/pie label 等关键用例固化基准 | `test/runTest/` |
| 低 | 修正 `pre-publish.js` 头注释、清理 `entryPath !== 'option'` 死分支与孤儿模板 | `build/` |
| 低 | 把 `build-i18n` 的正则替换升级为 AST/TS transpile（已有注释掉的 `ts.transpileModule` 方案） | `build/build-i18n.js:94-99` |
| 低 | 为 `as any`（272 处）建立"只减不增"的 lint 基线 | `.eslintrc-common.yaml` |

---

## 14. 数据附录

### 14.1 源码分布（行数，按目录）

| 目录 | 行数 | 占比 |
|---|---|---|
| `src/component` | 36,665 | 26.5% |
| `src/chart` | 34,461 | 24.9% |
| `src/coord` | 16,975 | 12.3% |
| `src/util` | 11,816 | 8.5% |
| `src/data` | 8,814 | 6.4% |
| `src/core` | 5,279 | 3.8% |
| `src/model` | 4,302 | 3.1% |
| `src/scale` | 3,768 | 2.7% |
| `src/label` | 3,054 | 2.2% |
| `src/visual` | 2,204 | 1.6% |
| `src/animation` | 2,198 | 1.6% |
| `src/layout` | 1,087 | 0.8% |
| `src/export`(1,126) + `src/i18n`(3,902) + `src/preprocessor`(628) + `src/view`(464) + `src/processor`(382) + `src/theme`(329) + `src/legacy`(161) + `src/renderer`(50) + `src` 根入口(712) | 7,754 | 5.6% |
| **合计** | **≈138,500** | 100% |

注：`src` 下 `.ts` 文件共 596 个（按 `Get-Content` 逐文件统计 138,546 行）。若排除 `src/i18n`（27 种语言的文案数据）与纯类型/入口文件，核心实现约 11 万行。

### 14.2 扩展点与注册密度

| 扩展点 | 调用点 |
|---|---|
| `registerAction` | 54 |
| `registerLayout` | 28 |
| `registerVisual` | 23 |
| `registerProcessor` | 17 |
| `registerCoordinateSystem` | 10 |

### 14.3 优先级常量（`src/core/echarts.ts:158-209`）

```
PROCESSOR: SERIES_FILTER 800 → DATASTACK 900 → AXIS_STATISTICS 920 → FILTER 1000
           → DEFAULT 2000 → STATISTICS 5000
VISUAL:    LAYOUT 1000 → PROGRESSIVE_LAYOUT 1100 → GLOBAL 2000 → CHART 3000
           → COMPONENT 4000 → CHART_DATA_CUSTOM 4500 → POST_CHART_LAYOUT 4600
           → BRUSH 5000 → ARIA 6000 → DECAL 7000
```

### 14.4 默认阈值（`src/model/globalDefault.ts`）

| 配置 | 默认值 | 作用 |
|---|---|---|
| `animationThreshold` | 2000 | 数据量超过则关闭动画 |
| `progressiveThreshold` | 3000 | 超过则启用渐进渲染 |
| `progressive` | 400 | 每帧渲染条数 |
| `hoverLayerThreshold` | 3000 | 超过则收敛到单一 hover 层 |
| `stateAnimation` | `{duration:300, easing:'cubicOut'}` | 状态切换动画 |
| `useUTC` | false | 时间轴时区 |

### 14.5 主要入口文件

| 文件 | 包含 |
|---|---|
| `src/echarts.all.ts`（374 行） | 全部 23 图表 + 30 组件 + 5 feature + 2 渲染器 |
| `src/echarts.common.ts` | 常用子集 |
| `src/echarts.simple.ts`（46 行） | line/bar/pie + gridSimple + aria + dataset + canvas |
| `src/echarts.blank.ts` | 仅 `export * from './echarts'`（无内置能力） |
| `src/echarts.ts` | `export * from './export/core'` + canvas + dataset + labelLayout |

### 14.6 验证用的关键命令

```shell
npm run checktype     # tsc --noEmit + bmap 子项目
npm run lint          # eslint src / ssr/client/src / extension-src
npm test              # jest（test/ut/jest.config.cjs）
npm run test:dts:fast # 类型产物测试（不装依赖）
npm run dev           # esbuild watch + http-server 打开 test/
npm run release       # lib → i18n → dist → esm → extension → ssr
```

---

## 15. 结论

ECharts 是一个**把"声明式配置"编译为"可增量更新的图形树"的框架**，其架构精髓可归纳为四条：

1. **一套更新周期**（`EC_CYCLE`）统管所有入口（`setOption`/`dispatchAction`/`resize`/`lazyUpdate`/`appendData`），并严格规定"模型只能周期起点改、事件只能周期结束发"。
2. **一条阶段流水线**（processor → coordSys → visual → layout → render），用 `Scheduler` + `Task` 单链表 + 优先级实现可组合、可渐进、可 dirty 传播的数据流。
3. **一张注册表**（`echarts.use` + 类管理器 + 10 类 registerXXX），让 23 图表 × 23 组件 × 8 坐标系能自由组合，同时通过 `ExtensionAPI` 白名单守住内核边界。
4. **一切为性能让路，但代价被显式记录**：索引不复制、typed array、large/渐进/hover 层、维度裁剪，以及随处可见的"为性能不封装"注释。

它的主要技术债集中在**类型严格性（未开 strictNullChecks + 272 处 `as any`）**、**核心单体文件（`echarts.ts` 3,434 行）**、**模块级全局单例**、**隐式契约（优先级/维度名/dependencies）缺少注册期校验**，以及**构建链的文本处理脆弱性**。测试体系三层齐备（单测/视觉/类型），但生产分支与 CI 覆盖存在明显空档。

对使用者：理解 `EC_CYCLE` 与 `PRIORITY` 这两张表，就能预测绝大多数"为什么配置没生效 / 顺序不对 / 性能差"的问题。
对贡献者：理解"实现 → `src/export/*` → `src/echarts*.ts`"三处同步，以及 `install(registers)` 契约，是提交可用 PR 的最小知识集。
