# ct-client-found v2：index-inject 事件线 + SPA 壳 + CT_SEED 真库

状态：🟢 已实现（随 `@cordis-tavern/ct-client-found@0.0.2-rc` 源码落地；**`distIndex` 服务已删除**）

> 来源演进：原设计「通过 ctx.provide 注册 distIndex」→ **做减法改为** ct-index-renderer 事件线；并入 SPA 壳与 CT_SEED 完整化。
> 一期发现 / 拓扑 / BOOT / 懒加载契约不动。宿主路由见 [[ct-host-web-v2]]；index 插入正典见 [[ct-index-renderer]]；未具名回退见 [[ct-frontend-static]]。

## 1. 目标与范围

| 项 | 结论 |
| --- | --- |
| ~~`distIndex` 服务~~ | **已删除**（本包不再 `ctx.provide('distIndex')`） |
| index 注入 | 经 **`index-inject`** 事件线：发布序固定四段，renderer 控制插入 |
| SPA 壳 | `index.html` 内置 JS：经 CT_SEED 取 `react-router-dom`，表内匹配 + history 监听，**无默认路由** |
| CT_SEED 完整 | `react` / `react-dom/client` / `react-router-dom` 真库工厂；库在 **`dependencies`**，随插件下载后打进 `dist/seed` |
| 明确不做 | 本包 provide `distIndex`、服务端读盘吐 HTML、静态 MIME 直出、内置 `*` / index 路由 |

## 2. `index-inject` 事件线（取代 distIndex）

**依赖**：`export const inject = ['webServer', 'indexRenderer'] as const`。

**发布**（`publishIndexInject`，固定四条，≠ 浏览器执行序）：

| # | placement | kind | content 要点 |
| --- | --- | --- | --- |
| 1 | `head` | `inline` | `window.__CT_SEED__=…` + runtime IIFE（CJS 懒加载表） |
| 2 | `head` | `script` | 每个 `immediately` 模块一条 `<script src="{url}">`（本地回环提前下载） |
| 3 | `body` | `script` | `__CT_MODULES__.load` 本包自身（`SELF_ID`，`getFactory` 防重） |
| 4 | `global` | `global` | `window.__CT_BOOT__=…`（**无** head/body 位置） |

```ts
ctx.emit('index-inject', { kind, placement, content })
```

**主页组装**：

1. `stripIndexStubs(readTemplate())`——剥 stub SEED/BOOT 与外部 boot-runtime；
2. `ctx.indexRenderer.apply(html)` → `homeHtml`（`global` 前缀文档最前，再 head、再 body；同 placement 保发布序）；
3. exact `homepage` 路由 `res.end(homeHtml)`。

无 `indexRenderer`（单测/未装载）：本地 `localAssembleHome` 按同序拼齐并 `logger.warn`。

**不 provide**：无 `distIndex` 导出、无 `distIndexPath` 对外 API。renderer 自有的 `distIndex()` 返回的是 **renderer 包**的 `index.html` 路径，与本包无关。

## 3. SPA 壳（`index.html` 内置）

```js
const off = window.__CT_SPA__.register('/history', HistoryPage)
off()
```

1. `boot-runtime` 消费 SEED → `ready`（BOOT 已由 `global` 片段就位）；
2. `require('react' | 'react-dom/client' | 'react-router-dom')`；
3. 挂载 `#root`：`BrowserRouter` + 仅注册表项的 `Routes`。

| 时机 | 行为 |
| --- | --- |
| 首屏 | `BrowserRouter` 读 `location.pathname` 在表内匹配 |
| 后续 | 库内监听 `popstate` / history（含 `pushState` / `<Link>`）后重匹配 |
| 空表 / 无匹配 | **不**内置默认路由 |

## 4. CT_SEED 完整与「随插件下载」

1. `dependencies`：`react` / `react-dom` / `react-router-dom`（**内核五库禁止进 dependencies**）；
2. `pnpm build` esbuild → CJS → `function(){…}` → `dist/seed/{react,react-dom-client,react-router-dom}.js`；
3. `seedTable()` 读工厂；**缺文件或非 function → 抛错**，不回落 stub；
4. 经事件线 **head 内联**进主页（不进 BOOT、不二次下载）。

**契约检查**：`dependencies` 含三件套；`dist/seed/*` 存在且长度 > 500；内核不在 `dependencies`。

## 5. 与 frontend-static 的分工

```
未具名 GET
  → host 无 exact/prefix 命中
  → ct-frontend-static fallback：200 + location.replace(homepage)（不读文件）
  → 浏览器落到 exact homepage（client-found 经 index-inject 组装的 HTML）
  → react-router-dom 按 pathname 在 __CT_SPA__ 表内分发
```

## 6. 验收（实现口径）

1. 本包 **不** `provide('distIndex')`；`inject` 含 `indexRenderer`；
2. `publishIndexInject` 固定四条，placement 序为 `head, head, body, global`；
3. `seedTable()` 三键为真库工厂，形状正确；
4. `index.html` 含 `__CT_SPA__` / `register` / `require('react-router-dom')`，**无** `path: '*'`；
5. `check-contract` + 全仓测试通过。
