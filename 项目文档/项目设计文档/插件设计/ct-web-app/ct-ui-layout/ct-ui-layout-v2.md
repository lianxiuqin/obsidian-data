---
tags:
  - 插件开发
  - overlay
  - 前端渲染
---

# ct-ui-layout v2（overlay 悬浮层）

状态：🟢 已实现（`0.3.0-alpha.1`；overlay 悬浮层，源码 `cordis-tavern-plugin/ct-ui-layout/src/overlay.ts` + `overlay-mount.ts` + `src/client/entry.ts`）

v2 在 [[ct-ui-layout]]（v1 舞台布局）基础上**新增一层 overlay**：一个覆盖整个视口的悬浮层，各插件可自由向内注册 React 组件（**桌面小组件式**）。悬浮组件可**实时拖动**、**位置持久化记住**，且**保留组件自身全部交互**（点击、输入等）。v1 的 root / topbar / main 槽位结构不变。

## 目标

- 新增 `overlay` 悬浮层，属性为 **`list`**：多个插件各自注册 React 组件，全部保留、按 `order` 罗列。
- 悬浮组件**可拖动到视口任意位置**（含 45% 舞台之外的区域）。
- 拖动**实时跟随**指针；拖完的**位置记住**（ct-persist 持久化，重启后回到原位）。
- 组件内部交互**不受拖动影响**：按钮、输入框等可交互元素上的按下**永远触发组件自身的点击/输入，不触发拖动**；只有**不可交互区域**才能拖动组件。
- 点击与拖动的区分：**位移阈值为主**（按下即进入待定，位移超过阈值才算拖动；原地松手即点击），不采用长按计时。
- **触摸进 v2 范围**：手势仲裁**内部滚动优先**——组件内可滚动区优先滚动，可交互元素优先点击，仅不可交互区域可触摸拖动。
- 多个悬浮组件重叠时，**被按下（点击或拖动）的组件置于最上层**，桌面小组件式直觉。

## 非目标（v2 不做）

- **调整大小**：不做运行时 resize。组件尺寸**完全由组件作者决定**（组件内容自撑），overlay 只提供层级与渲染。
- 吸附网格、自动排布、避让。
- 隐藏/显示开关、组件商店、跨窗口同步。
- 非 React（DOM/HTML）组件注册。
- 拖动把手（handle）自定义：v2 只做「可交互元素识别」，不做作者声明式拖动区。

## 概念模型

对标**手机桌面小组件**：

| 桌面小组件 | overlay 对应 |
|---|---|
| 小组件自由摆放 | 注册的 React 组件可拖到视口任意位置 |
| 位置自动记住 | ct-persist 持久化 `{ x, y }` |
| 点按图标打开 App | 组件内可交互元素正常点击 |
| 按住空白处拖动 | 不可交互区域按下并位移才拖动 |
| 被点的小组件浮到最上 | pointerdown 置顶 |

## 层级结构

overlay 是**独立于舞台的一层**，覆盖整个视口（`position: fixed`），不参与 v1 的 flex 布局：

```
body
├── .ct-ui-layout / .ct-ui-layout-stage      v1 舞台（root/topbar/main），不变
└── .ct-ui-layout-overlay                    overlay 层：position: fixed; inset: 0
    └── .ct-ui-layout-overlay-item × N       每个注册组件一个壳（Shell）
        └── 注册的 React 组件                 尺寸自撑（fit-content），内容与交互归组件作者
```

| 项 | 值 |
|---|---|
| 挂载方式 | **自挂载独立 React 根**：client entry 用 `createRoot`（`react-dom/client` 取自 `__CT_MODULES__`）把 overlay 层挂到 body 下自己的容器，**不活在 Layout 树里**——root 槽将来被更高 priority 接管时悬浮层不受牵连 |
| overlay 层 | `position: fixed; inset: 0; pointer-events: none`（不挡下层）；**`z-index: 10000`**（盖过舞台内容） |
| 壳（Shell） | `position: absolute`；`pointer-events: auto`（自身可交互）；`transform: translate(x, y)` |
| 壳的定位 | 只存 `{ x, y }`（视口坐标），不存 width/height（尺寸归组件作者） |
| 拖动实现 | Pointer Events（pointerdown / pointermove / pointerup），鼠标与触摸统一；触摸手势仲裁见下节 |
| 拖动渲染 | 拖动中改 `transform`，不触发组件重挂载，**组件状态不丢** |

> 挂载方式与 z-index 是设计者定夺（评审问题 1）：自挂载是为了悬浮层生命周期独立于 `root` 槽的胜出者；若未来改回 Layout 树内渲染，需接受 root 接管会连带删除悬浮层。

## 插槽契约（属性：`list`）

overlay 是一个 **`list` 插槽**，槽名 `overlay`。业务插件经 ct-ui-slots v2 的组合式签名注册：

```ts
ctx.slots.register(
  { name: 'overlay', id: 'my-widget', kind: { list: true }, order: 0 },
  MyWidget,
)
```

| 字段 | 说明 |
|---|---|
| `name` | 固定 `'overlay'`（分段式命名下的一级槽名） |
| `id` | 组件 id；同槽重复 → 告警并按 ct-ui-slots v2 规则取 `order` 小者。**同时是位置持久化的键** |
| `kind` | `{ list: true }`（只此一键） |
| `order` | int，小者靠前；仅影响初始罗列/叠放次序 |

> **持久化键 = 注册 `id`**，因此同一 `id` 在卸载重装后仍回到原位置。
> 壳需要拿到 `id`：ct-ui-slots 渲染 list 条目时把 `{ id, order }` 作为 `entry` 注入组件 props（对现有注册方无破坏），壳从 `entry.id` 取键。

消费：overlay 层调用 `renderSlot('overlay', {})` 拿到组件元素数组，**逐个包壳**（`Shell` 包住每个元素），壳负责拖动、置顶、位置读写；**组件自身不需要写任何拖动代码**。

**空槽与误声明的契约（本地防御，不动 ct-ui-slots）。** ct-ui-slots 的 `renderSlot` 在「槽未登记」与「list 槽零组件」时**抛错**，而零悬浮组件恰是系统常态。为此 overlay 消费侧自行兜底：

| 情形 | 行为 |
|---|---|
| 槽未登记 / 零组件 | 渲染空 overlay 层，**不报错、不占位壳**（不学 v1 topbar/main 的可见占位） |
| 误注册为 `single`（漏写 `kind: { list: true }`） | 告警一次，宽容处理：单组件也包一个壳渲染，不崩 |
| `renderSlot` 返回非数组 | 同上告警 + 宽容处理 |

> 只改 ct-ui-layout 消费侧（try/catch + 形状校验），**不改 ct-ui-slots 的已实现契约**——协调改包的发版成本与回归风险大于本地兜底。

## 拖动与点击判定（两层防护）

**第一层：可交互元素识别（交互优先）。**

按下事件落在可交互元素上（含其子树）时，**直接交还组件**：不进入拖动态，点击/输入照常。识别清单（默认可交互元素）：

```
button, input, select, textarea, a, label, summary,
[role="button"], [role="link"], [role="checkbox"], [role="switch"], [role="tab"],
[contenteditable="true"], [data-ct-no-drag]
```

判定方式：`event.target.closest(selector)` 命中即可交互。`[data-ct-no-drag]` 是通用豁免口（作者给自绘控件标注），属于识别清单的一部分，不是独立的声明式拖动区机制。

**触摸手势仲裁（内部滚动优先）。** 触摸上「拖组件」与「滚组件内部」会互抢手势，v2 明确**内部滚动优先**：

| 触摸落点 | 行为 |
|---|---|
| 可交互元素（第一层命中） | 交还组件：点击/输入照常，不拖 |
| 可滚动内容区（`overflow: auto/scroll` 且实际溢出） | 交还浏览器滚动：不拖。判定：`event.target.closest(...)` 沿途找到可滚动祖先即让位 |
| 其余不可交互区域 | 拖动；本次手势抑制浏览器默认滚动，防止手势被抢 |

实现口径：`touch-action` 不设在 overlay 层级或壳整体，而是**按下时才对命中路径之外的壳生效**——具体做法：壳默认 `touch-action: auto`，pointerdown 命中「可拖区域」时对本次手势抑制默认滚动（`preventDefault` 于 `touchstart`/`pointerdown` 可拖路径），命中滚动区/交互区则完全不干预。鼠标无此问题（无浏览器手势抢占）。

**第二层：位移阈值（点击 vs 拖动）。**

按下落在**不可交互区域**时进入**待定**态：

| 事件 | 行为 |
|---|---|
| pointerdown | 记录起点；壳置顶（见下）；不立即判拖动 |
| pointermove（待定中） | 位移 **≥ 5px**（`DRAG_THRESHOLD`）→ 转入拖动态；此后组件的 `click` 被抑制 |
| pointermove（拖动中） | 实时更新 `transform`，指针捕获（`setPointerCapture`） |
| pointerup（未超阈值） | 视为**点击**，事件原样交给组件 |
| pointercancel | 回滚到按下前位置（本次拖动视为未发生） |

> 阈值 5px 是防手抖的「位移」判定，不是时间判定；**点击不需要等长按**，拖动也不卡手。若实测手抖误触，再考虑加最小按压时长（~100ms）辅助，v2 不做。

**两层叠加**：可交互元素上永远不拖；不可交互区域上「移动了才拖、没移动就是点击」。

## 层级：交互即置顶

- 壳上 `pointerdown`（无论最终是点击还是拖动）→ 该壳**升到最上层**。
- 实现：overlay 层维护递增 `zIndex` 计数器，按下时赋 `++counter`；`order` 只决定初始叠放次序。
- 不提供手动置顶/置底 API（非目标）。

## 位置持久化（ct-persist）

| 项 | 值 |
|---|---|
| 存储 | `ctx.persist.json`（ct-persist，`data/` 固定前缀） |
| namespace / key | `namespace: 'ui-layout'`，`key: 'overlay-positions'` |
| 数据形态 | `{ [组件 id]: { x: number, y: number } }`（视口坐标，壳左上角） |
| 写入时机 | 拖动结束（pointerup 且发生过位移）落盘一次；拖动过程中不写 |
| 读取时机 | overlay 层挂载时整表读入；每个壳按 `entry.id` 取自己的 `{ x, y }` |
| 无记录 | 首次出现的组件落默认位（视口左上 `(24, 24)` 起按 `order` 阶梯偏移，步长 32px） |
| 越界回正 | **拖动结束时**、加载后与窗口 resize 时，把壳钳回视口内可见区（保证至少一角可见，组件不会飞出屏幕外找不回来）；落盘的是钳制后的位置 |
| 传输 | 浏览器侧经 webServer 读写（同 ct-ui-theme 的「persist 持久化 + webServer 传输」模式）；Node 侧 `inject: ['webServer', 'persist']` |

## API 契约汇总

对业务插件（浏览器侧）：

```ts
// 注册悬浮组件（唯一入口，走 ct-ui-slots v2 签名）
ctx.slots.register(
  { name: 'overlay', id: 'my-widget', kind: { list: true }, order: 0 },
  MyWidget,   // React 组件，尺寸自撑；props 含 entry: { id, order }（由 slots 注入）
)
```

组件作者职责（**归作者**）：内容、尺寸、内部交互、样式。
overlay 层职责（**归本插件**）：壳、拖动、点击/拖动判定、置顶、位置持久化、越界钳制。

## 包形态

| 项 | 值 |
|---|---|
| 包名 | `@cordis-tavern/ct-ui-layout`（同一包演进，不另立包） |
| 版本 | 设计稿；实现时升 `0.3.0-alpha.1` |
| 槽位结构 | root / topbar / main / **overlay**（新增一个，其余不变） |
| Node `apply` | 由空 `apply` 变为注册 persist 读写端点（`inject: ['webServer', 'persist']`） |
| 前端 `entry.ts` | 新增 overlay 壳渲染与拖动逻辑；`inject: ['slots']` 不变 |
| 导出 | 现有 `createLayoutComponent` / `BREAKPOINT` 之外，新增 `createOverlayComponent` / `DRAG_THRESHOLD` |

## 验收

1. `overlay` 槽为 `list`：两个插件各注册一个组件，**同屏罗列**，互不覆盖内容。
2. 按住组件**不可交互区域**并移动 ≥ 5px → 组件实时跟随指针；松手后停在新位置。
3. 在同一位置按下松开（位移 < 5px）→ 组件自己的 `click` 正常触发，不发生拖动。
4. 点击组件内的 `button` / 输入框 → 按钮点击、文本输入正常，**不触发拖动**。
5. 拖动后刷新/重启 → 组件回到拖后位置（persist 回读）。
6. 两个组件重叠 → 点击或拖动下面那个，它升到最上层。
7. 窗口缩小到把组件甩出视口 → 组件被钳回可见区。
8. **零悬浮组件**（无插件注册 overlay）→ 空 overlay 层，无报错、无空壳。
9. **触摸**：可交互元素上触摸 → 点击照常；组件内部可滚动区上触摸拖 → **内容滚动**（不拖壳）；不可交互区域上触摸拖 → 壳跟随手指。
10. **拖出视口**：拖到屏幕外松手 → 壳钳回可见区，落盘钳后位置。

## 留给实现者

- `entry` 注入的具体形状（`{ id, order }` 还是平铺进 props）与 ct-ui-slots 的最小改动方式。
- 落盘防抖间隔（拖动结束即写即可，如需合并连拖再加 ~200ms 防抖）。
- 默认位阶梯偏移公式与钳制边界（保证「至少一角可见」的几何算法）。
- `DRAG_THRESHOLD = 5` 的实测微调。

## 总结

ct-ui-layout v2 在 v1 舞台之上新增 **overlay 悬浮层**（`list` 插槽）：各插件注册 React 组件（桌面小组件式），组件**可拖到视口任意位置**、**位置经 ct-persist 持久化**、**重叠时交互即置顶**。点击与拖动以**位移阈值**（5px）区分，且**可交互元素识别**在前——按钮/输入框永远优先触发组件自身交互，只有不可交互区域才能拖；触摸上**内部滚动优先**。组件尺寸与内容全归组件作者，overlay 只做**层级与渲染**。
