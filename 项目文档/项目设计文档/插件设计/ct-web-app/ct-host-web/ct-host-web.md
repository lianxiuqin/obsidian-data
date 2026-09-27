---
tags:
  - 插件开发
  - host
  - HTTP服务
  - CIDR
---

# ct-host-web

状态：🟢 已实现（0.0.2-rc；最简版 + **webServer 路由 / fallback** 已落地）

> 现状：插件仓库 `cordis-tavern-plugin/ct-host-web/` —— HTTP 服务 + 配置四键（含 CIDR 白名单）+ **路由注册 / fallback**（服务 `webServer`：`hostRegister` / `fallbackRegister`，见 [[ct-host-web-v2|v2]]）。
> 共性概念与通用对接约定见 [[项目文档/项目设计文档/插件设计/插件概念总览|插件概念总览]]。

**包名**：`@cordis-tavern/ct-host-web`

**功能**：实现 **host 服务**的开启，作为基础插件对外提供 **HTTP 服务**能力，供其他插件注册路由。

**实现方式**：通过 **Node.js** 的 `node:http` 内置模块实现，不依赖第三方框架。

**服务**：

| 服务名 | 内容 | 可见时机 |
| --- | --- | --- |
| `webServer` | `{ hostRegister, fallbackRegister }` | apply 同步段 |
| `host` | `{ name: 'host', server }` | 监听成功后 |

消费方注册路由用 `inject: ['webServer']`。

**配置项**（英文键为准）：

| 配置项 | 说明 |
|---|---|
| `listen` | 控制是否监听外部请求。为 `true` 时监听 `0.0.0.0`（所有网卡地址）；缺省 `false`（只绑 `ip`） |
| `ip` | 监听地址，仅在 `listen` 为 `false` 时生效，默认 `127.0.0.1` |
| `listenWhitelist` | 仅在 `listen` 为 `true` 时生效。数组，支持 CIDR；生效时只接受白名单内 IP 的连接。默认 `['127.0.0.1']` |
| `port` | 服务监听端口，默认 `3100`（`0` = 系统分配） |

> 未知配置键抛 `TypeError` 并指名键名。

**运行行为**：服务启动后，自动打开浏览器访问 `http://127.0.0.1:<port>`（默认 `http://127.0.0.1:3100`）。

**src 布局**：`src/index.ts`（插件入口）、`src/server.ts`（HTTP 服务）、`src/router.ts`（路由表）、`src/config.ts`、`src/whitelist.ts`、`src/open.ts`。
