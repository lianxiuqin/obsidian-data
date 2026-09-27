---
tags:
  - 前端插件
  - 懒加载
  - 拓扑排序
  - SPA
  - index-inject
---

# ct-client-found：前端插件发现与加载

状态：🟢 已实现（`@cordis-tavern/ct-client-found@0.0.2-rc` 已发布 registry；插件仓库 `cordis-tavern-plugin/ct-client-found/`）

> 现状：浏览器内核 + Node 插件包均已落地。实现 spec 见插件仓库 `ct-client-found/docs/ct-client-found-设计.md`。
> 本页为**设计正典**（应该是什么）；字段读取方与发布细节以实现为准。
> **index 注入已解耦**：本插件**不** provide `distIndex`；启动时段按固定发布序 `emit('index-inject', …)`，由 [[ct-index-renderer]] 收集并 `apply` 进主页 HTML。SPA 壳 / CT_SEED 真库见 [[ct-client-found-v2]]；共性概念见 [[插件概念总览]]；`ct.client` 见 [[插件契约层]]。

## 定位

本插件的职责是**识别前端插件**，并把它们组装成一张**可以正常加载的列表**：通过 `ct.client` 自定义字段识别前端插件，依据依赖关系做**拓扑排序**得到加载次序，最终以注入 HTML 的方式向浏览器提供加载顺序。

本插件是**发现并加载前端页面的最基础插件**，可用作前端内核；消费 host 的 `webServer` 注册页面与 bundle 路由；消费 [[ct-index-renderer]] 的 **`indexRenderer`** 完成 index 片段插入。

**配套落地**：未具名路由的 SPA 回退由 [[ct-frontend-static]] 申领 host `fallback`（不读文件）；回退壳 `location.replace(homepage)` 后由本插件主页 HTML + 客户端 `react-router-dom` 完成分发。

## 识别与排序：`ct.client` 自定义字段

插件通过 `package.json` 的 `ct.client` 扩展字段声明前端身份与加载语义：

### `ct.client.platform`：前端插件标记

当取值为 `web` 时，将其标记为**前端插件**并记录。存在 `ct.client` 即表示含前端结构。

**配套入口（正典）**：浏览器 bundle 路径一律由 **`exports["./client"]`** 声明（标准 npm `exports` 子路径，不是 `ct` 字段）。`discover` 读该字段得到 `clientPath`；有 `platform: "web"` 却缺 `./client` → `TypeError` 指名包。字段定义见 [[插件契约层]]。

### `ct.client.inject`：依赖声明

一个由**包名组成的数组**，用于声明前端依赖，是**拓扑排序**的主要证据。可缺省为 `[]`。悬空依赖（不在入选前端集合内）→ **抛错**。前端依赖不参与后端加载排序（后端顺序由 Cordis 的 `inject` 决定，见 [[启动层]]）。

### `ct.client.immediately`：加载策略

**布尔类型，必填无默认**，用于判断插件是否需要**立即注册工厂函数**，是浏览器**懒加载**条件的主要证据：

- `true`：启动时立即注册工厂函数；
- `false`：不注册工厂函数，调用时才注册，并将注册与实例化（物化）一起进行。

缺字段或非布尔 → `TypeError` 指名包与键。

## index 注入：`index-inject` 事件线（解耦后）

主页 HTML **不再**由本插件直接 `injectHtml` 写死 SEED/BOOT，而是：

1. `inject: ['webServer', 'indexRenderer']`——renderer 已 ACTIVE、事件订阅已挂；
2. apply 内 `publishIndexInject(emit, …)` **按固定发布序** `ctx.emit('index-inject', payload)`；
3. `stripIndexStubs(template)` 剥掉模板 stub 后，`ctx.indexRenderer.apply(html)` 得到 `homeHtml`；
4. exact `homepage` 路由吐出该 `homeHtml`；无 renderer 时本地按同序拼齐（告警）。

### 发布序（≠浏览器执行序）

| # | placement | kind | content |
| --- | --- | --- | --- |
| 1 | `head` | `inline` | CJS 懒加载表：`__CT_SEED__` + runtime IIFE |
| 2 | `head` | `script` | 本地回环提前下载：immediately 批 `<script src="url">` |
| 3 | `body` | `script` | 向 CJS 表 `load` 本包自身（`getFactory` 防重） |
| 4 | `global` | `global` | `window.__CT_BOOT__`（**无** head/body 位置） |

**发布序 ≠ 执行序**：`global` 由 renderer **前缀到文档最前**，保证 `__CT_BOOT__` 先于 head 内 runtime 存在；index 被浏览器解析后，`__CT_BOOT__` 仍控制 bundle 下载。同 placement 内保持收集（发布）顺序。

事件名固定 **`index-inject`**（与 [[ct-index-renderer]] 同值，不跨包 import）。载荷契约与插入规则见 [[ct-index-renderer]]。

## 组合与注入：BOOT 启动地图

所有插件组合后的加载表统一命名为 **`window.__CT_BOOT__`**，以 `placement: 'global'` 的 `index-inject` 片段写入（不是 head 内联）。JSON 序列化时 `<` 转义为 `u003c`，防 `</script>` 闭合。BOOT 内 `url` 为**相对路径**（`bundlePrefix + id + /bundle.js`），同源下载，不绑死 port。

**注入卫生**：`stripIndexStubs` 剥掉模板里 stub `__CT_SEED__` / `__CT_BOOT__` 与外部 `<script src="…/boot-runtime.js">`，避免与事件线双跑。

BootMap 形状：

```js
window.__CT_BOOT__ = {
  order: ['pkg-a', 'pkg-b'],      // 拓扑序（成环段按发现序追加）
  degraded: ['pkg-x'],            // 成环退化 id，可为 []
  modules: {
    'pkg-a': { url: '/bundles/pkg-a/bundle.js', immediately: true, deps: [] }
  }
}
```

## 懒加载机制：cjs 工厂函数表

**工厂函数名即包名**。插件通过模拟 Node 的**模块化懒加载**机制（CommonJS 语义），首次加载时经由 cjs 列表完成。

cjs 表（工厂函数登记表）由插件提供并保存在**内存**中，全局名称为 **`window.__CT_MODULES__`**：

- 通过 **`window.__CT_MODULES__.load({ id, factory })`** 登记工厂函数；重复 id → `console.error` 后忽略（不覆盖）
- 浏览器运行时通过 `window.__CT_BOOT__` 下载 bundle 时，插件**主动调用 `load()` 向 cjs 表登记**
- `immediately = false` 的插件**不主动注册**
- 被 `require` 引用时**优先查阅工厂函数表**并执行**物化**（调 factory 一次并缓存 exports）
- 对 `false` 插件：被引用时先执行已缓存文本完成注册、再物化

**同步 `require` 硬约束**：

| 约束 | 说明 |
| --- | --- |
| bundle 格式 | **IIFE / 经典 script**，不是 ESM |
| 预取 | `fetch` → **`await res.text()`** → 内存 `Map<id, string>`；不缓缓存 Response |
| `require` | **严格同步**：只对已注册工厂生效；未注册 → throw；**不**现场下载、不返回 Promise |
| 等待口 | **`ready` Promise**：true 批注册完成 + false 批文本预取完成后的唯一等待点 |

### 基础工厂函数预登记（`__CT_SEED__`）

cjs 表在创建时消费 **`window.__CT_SEED__`** 预登记基础工厂，以减少 **IO 开销**。SEED 经事件线 **`head` 内联**与 runtime 同一脚本写入，不进 `modules`、不二次下载。

**键（完整形态）**：

| 键 | 来源 |
| --- | --- |
| `react` | `package.json` `dependencies` → 构建打成 `dist/seed/react.js` 工厂 |
| `react-dom/client` | 同上 → `dist/seed/react-dom-client.js` |
| `react-router-dom` | 同上 → `dist/seed/react-router-dom.js` |

`seedTable()` 从 `dist/seed/*.js` 读 **function 工厂源码**；缺文件 → **抛错**（不静默回落 stub）。库本体随插件 **npm dependencies 下载**，构建时打进 seed，见 [[ct-client-found-v2]]。

## SPA 壳与客户端路由（v2）

`index.html` **内置**一段 SPA 壳脚本（模板与 `dist/index.html` 同步；生产经事件线保留模板壳、剥掉 stub 表）：

1. 等待 `__CT_MODULES__.ready`，`require` CT_SEED 三件套；
2. 业务插件经 **`window.__CT_SPA__.register(path, component)`** 登记路由表（返回 disposer）；
3. 挂载 `BrowserRouter` + 动态 `Routes`：**首屏**读 `location.pathname` 在表内匹配；**后续**由 `BrowserRouter` 监听 `popstate` / history 变更重分发；
4. **永不内置** `*` / index 默认路由；空表 → 空渲染。

服务端未具名路径不读盘：由 [[ct-frontend-static]] 回退壳 `location.replace(homepage)` 落到本插件 exact 主页后，再走上述客户端分发。

## 对外服务：无 `distIndex`（做减法）

**已删除** `ctx.provide('distIndex', …)` 与对外 `distIndexPath()`。index 磁盘路径与片段插入职责全部归 [[ct-index-renderer]]（renderer 自有 `distIndex` 可返回**其** `index.html` 路径，与本插件无关）。

本插件对外消费面仅为：

| 面 | 说明 |
| --- | --- |
| `inject` | `['webServer', 'indexRenderer']` |
| 发布 | `ctx.emit('index-inject', payload)`（固定四段，见上） |
| 路由 | host `hostRegister`：exact `homepage` + prefix `bundlePrefix` |

## 内核定位

cjs 表由本插件直接提供。本插件是**发现并加载前端页面的最基础插件**，可用作内核。

主 `index.html` 承载 SPA 壳；SEED/BOOT/回环 script 经 [[ct-index-renderer]] 事件线插入。**Vite 由本包 devDependency 自装**。

## 加载时机

1. `immediately = true` 的插件**优先加载**（事件线亦对这批发 head `<script src>` 提前下载）；
2. `true` 完成后，**立即批量下载**所有 `false` 插件的 bundle 文本，但**不注册工厂函数**——注册与物化推迟到被引用时（先 `await ready`，再同步 `require`）。

## 依赖死锁处理

若依赖关系**成环**，环中所有插件退化为**直接加载**（整环标记 `degraded`，按发现序排入 `order` 末尾，不抛错）。开发者会提供基础插件，保证即使不依赖任何外来插件，系统也能正常运行。

## 发现来源（Node 侧）

对齐启动层：调用解耦后的 **`assembleTable`**（`@cordis-tavern/cordis-tavern` 导出）装配最终插件表，再按包名在 `profile/<env>/node_modules` → `$CT_HOME/node_modules` 定位；**不用裸 `ct.profile`**。测试可注入表 / `assembleTable`。

## 装载与发布

| 项 | 值 |
| --- | --- |
| 包名 | `@cordis-tavern/ct-client-found` |
| 版本 | **0.0.2-rc**（已发布 registry，`latest`） |
| insert id | `clientFound` |
| `ct.bundle.path` | `./dist/cordis.patch.yml`（发布物仅 `dist/`） |
| `files` | `["dist/"]`（含 `dist/seed/*`） |
| 消费服务 | `inject: ['webServer', 'indexRenderer']` |
| 对外提供 | **无**（不 provide `distIndex`） |
| 发布 | `pnpm pack` / `pnpm publish`（或 `npm publish <tgz>`） |
| `dependencies` | `react` / `react-dom` / `react-router-dom`（**随插件下载**；内核仍只 peer） |
| peer | 内核五库 + `@cordis-tavern/cordis-tavern`（`assembleTable`） |

## 配置

| 键 | 类型 | 默认 | 说明 |
| --- | --- | --- | --- |
| `homepage` | string | `/` | exact 主页路径（事件线组装后的 HTML 由此路由吐出） |
| `bundlePrefix` | string | `/bundles/` | bundle 下载前缀（须以 `/` 开头） |

未知键与非法值 → `TypeError` 指名键名。

## 总结

ct-client-found 通过 `ct.client` 识别前端插件并拓扑排序，生成 **`window.__CT_BOOT__`** 加载表；浏览器端经 **`window.__CT_MODULES__`** 懒加载（IIFE、预取存文本、`require` 严格同步、`ready` 唯一等待口）。index 注入与本包解耦：**按固定发布序 `emit('index-inject')` 四段**（CJS 表 head 内联 → 回环 `script src` → body 自注册 → `global` BOOT），由 [[ct-index-renderer]] 插入主页；本包**不再** provide `distIndex`。CT_SEED 为 react 三件套真库（`dependencies` 随包下载）；`index.html` 内置 SPA 壳（`__CT_SPA__` + `react-router-dom`）；成环退化直载 + 基础插件保底解决依赖死锁。
