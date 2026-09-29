# ct-client-cordis

状态：🟢 已实现（`0.1.0-rc.1`）

## 目标

为**浏览器侧**提供加载插件能力：把 client bundle 物化出的插件装入**浏览器端 Cordis 实例**（完整 cordis 打包进浏览器），与 Node 内核实例各持一份。

## 职责

| 项 | 说明 |
| --- | --- |
| 物化 | 从 `__CT_MODULES__` 取 factory，调用得到插件对象 |
| factory 契约 | `{ name, apply }`（可含 `inject` / `provide`）。无 `apply` 视为身份 stub，跳过并 warn |
| 装载 | `new Context()` + `ctx.plugin(exports)`；按 **具体服务名** `inject` 依赖驱动 |
| 激活收尾 | 全部插件就绪后经 **`ctx.uiRenderer.onAllReady()`** 触发渲染（禁止 import / inject 具体渲染器，防依赖倒置） |
| 容错 | 物化 / apply 抛错 → `console.error` 指名 id，不拖垮其它插件 |
| 幂等 | 启动键挂在 `globalThis.__CT_CLIENT_CORDIS__`（IIFE 可能被 script/evalText 双执行） |

## 运行形态

- 浏览器侧装完整 cordis；Node 端空 apply 真装载（进 ct-client-found 发现链）。
- `exports["./client"]` 为装载器 IIFE；`ct.client`：`platform: web` / `immediately: true`。
- head 用 **`<link rel="preload">`** 预热，不重复执行 bundle（避免双开装载器）。

## 总结

ct-client-cordis 物化 `__CT_MODULES__` 工厂并装入浏览器端 Cordis；收尾调用 `ctx.uiRenderer.onAllReady()` 完成首屏渲染。
