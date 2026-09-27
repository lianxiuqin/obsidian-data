# ct-frontend-static

状态：🟢 已实现（**0.0.1-rc**；SPA fallback —— 未具名路由回退主页、HTTP 200，**不读文件**）

> 现状：插件仓库 `cordis-tavern-plugin/ct-frontend-static/`。共性概念与通用对接约定见 [[项目文档/项目设计文档/插件设计/插件概念总览|插件概念总览]]。
> 与 client-found 的分工：本插件只申领 host **fallback 槽**；真正吐 index + SPA 壳的是 [[ct-client-found|ct-client-found]]（exact `homepage`，HTML 经 [[ct-index-renderer]] 事件线组装）；客户端分发见 [[ct-client-found-v2|v2]]。本插件**不读文件**、不消费任何 index 磁盘路径服务（client-found 已删除 `distIndex` provide）。

**包名**：`@cordis-tavern/ct-frontend-static`

**功能**：fallback 回退路由（静态 MIME 直出本期不做）。

- **fallback 路由**：注册为 host 的 fallback 路由，host 匹配不到时交它处理
- **回退语义**：未具名路由 → 返回 **HTTP 200** 回退壳 → 浏览器 `location.replace(homepage)` 落到已注册主页；index 只是 **JS 启动器**，页面切换由动态路由库在客户端完成
- **不读文件**：不 `readFile`、不消费任何 index 路径服务做读盘

**依赖**：`inject: ['webServer']`；host 未就绪时本插件保持 PENDING（见 [[ct-host-web|ct-host-web]]）。

**配置**：`homepage`（默认 `/`）—— 回退目标，须与已注册 exact 主页一致。

**行为摘要**：

| 步骤 | 实现 |
| --- | --- |
| 申领 | `webServer.fallbackRegister(createFallback(homepage))` |
| 回退 | `200` + `text/html` 回退壳（`location.replace(homepage)`），**无读盘** |
| 卸载 | `ctx.effect` disposer 释放全局唯一槽位 |
