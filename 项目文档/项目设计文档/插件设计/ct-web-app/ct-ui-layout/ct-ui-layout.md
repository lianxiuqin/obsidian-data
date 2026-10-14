# ct-ui-layout

状态：🟢 已实现（`0.2.1-alpha.1`；双子插槽舞台布局，源码 `cordis-tavern-plugin/ct-ui-layout/`）

## 目标

提供**舞台式主布局**：接管 `root`，在中部门舞台内再开 `topbar` / `main` 两个插槽，供业务插件挂导航与主内容。  
（早期 V1 仅在 root 显示 `cordis-tavern-test` 验证整链，**已由本版取代**。）

## 布局结构（`src/layout.ts`）

```
.ct-ui-layout                    外层：flex 水平居中，高 100%，背景 --ct-color-bg
└── .ct-ui-layout-stage          内层舞台：高 100%，flex 列
    ├── .ct-ui-layout-topbar     导航区：flex 0 0 40px，背景 --ct-color-surface-2
    └── .ct-ui-layout-main       主内容区：flex 1 1 auto，背景 --ct-color-surface，overflowY: auto
```

| 项 | 值 |
|---|---|
| 舞台宽度 | **45%**（水平居中） |
| topbar 高 | **40px**（固定） |
| 响应式 | 窗口宽 **≤ 1000px** 时 `.ct-ui-layout-stage { width: 100% }`（`BREAKPOINT = 1000`） |
| media query | 由 `client/entry.ts` 注入 `<style id="ct-ui-layout-responsive">`（不重复注入） |

CSS 变量（缺省回退在括号内）：`--ct-color-bg`（`#181818`）、`--ct-color-surface`（`#1F1F1F`）、`--ct-color-surface-2`（`#262626`）。

## 插槽登记（`src/client/entry.ts`，浏览器侧）

factory：`{ name: 'ct-ui-layout', apply, inject: ['slots'] }`，经 `slots.register` 登记三个槽：

| 槽位名 | kind | 登记 id | 说明 |
|---|---|---|---|
| `root` | `{ single: true, priority: 1 }` | `ct-ui-layout` | 整页舞台组件 `Layout`（外层 + stage + 占位） |
| `topbar` | `{ list: true }` | `ct-ui-layout` | 导航区占位；**list 目前无消费方** |
| `main` | `{ single: true }`（priority 默认 0） | `ct-ui-layout` | 主内容区占位；`order: 0` |

**替换语义（对外契约）**

- `main` 为 **single**：同名再注册且 **`priority` 更大者胜出**，即业务插件可「以权重替换」主内容区。  
  例：`slots.register({ name: 'main', id: 'ct-test', kind: { single: true, priority: 1 } }, Memo)`。
- `topbar` 为 **list**：理论可多来源罗列；**当前无作用**，布局侧不消费。
- `root` 由本插件占据（priority 1）；业务插件不应改 root。

> `Layout` 在 topbar / main 区域内调用 **`props.renderSlot`** 挂出槽位内容（由 ct-ui-slots 注入）。
> `renderSlot('main')` 选出 single 胜出者（priority 大者），业务插件以更高 `priority` 注册即可「权重替换」主内容区。
> **槽位结构不变**：仍是 root / topbar / main。

## 与 slots 契约的对应

| 约定 | 本插件用法 |
|---|---|
| 注册签名 | v2 组合式：`register({ name, id, kind?, order? }, Component)` |
| single 胜出 | `priority` **大者优先**（`main` / `root`） |
| list | `order` 小者靠前（`topbar`，暂无消费） |
| React | `__CT_MODULES__.require('react')`，`createElement` 由 factory 在 `apply` 内绑定 |

## 包形态

| 项 | 值 |
|---|---|
| 包名 | `@cordis-tavern/ct-ui-layout` |
| 版本 | `0.2.1-alpha.1` |
| insert id | `uiLayout` |
| Node | 空 `apply`（只进装载链）；导出 `createLayoutComponent` / `BREAKPOINT` |
| 前端 | `exports["./client"]`；`ct.client`: `platform: web`，`immediately: true`，`inject: ["@cordis-tavern/ct-ui-slots"]` |
| factory | `{ name, apply, inject: ['slots'] }` |

## 验收（与 `test/layout.test.mjs` 一致）

1. `Layout` 树：外层 `.ct-ui-layout`（flex + `justifyContent: center`）→ `.ct-ui-layout-stage`（`width: 45%`）。
2. topbar：`flex: '0 0 40px'`、`height: 40px`。
3. main：`flex: '1 1 auto'`。
4. `BREAKPOINT === 1000`。

## 总结

ct-ui-layout 以 `root`（single/priority 1）接管整页，提供 45% 舞台（≤1000px 满页）+ **40px topbar** + **main** 双子插槽；业务侧可在 `main` 上以更高 `priority` 权重替换内容。topbar 为 list、当前无消费方。
