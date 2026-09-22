---
tags:
  - 插件开发
  - host
  - 路由
---

# ct-host-web v2

> 承接 [[项目文档/项目设计文档/插件设计/ct-host-web/ct-host-web|ct-host-web]] 的最简版现状（HTTP 服务 + 配置四键，路由注册 / fallback 未实现），本页描述 v2 设计：实现[[插件]]注册与 **[[兜底路由|fallback]]** 接口。

## 目的

实现插件的**注册**与 **fallback（兜底）** 接口。

## 插件注册

使用 `ctx.provide` 创建一个名为 **hostRegister** 的[[服务]]，共三个参数：`kind`、`path`、`handler`。

### kind：匹配模式

`kind` 决定匹配方式，分为两种：

- **prefix（前缀）**：使用[[前缀匹配]]。
- **exact（精准）**：使用[[精准匹配]]。

### path：路由路径

`path` 是以 `/` 为分隔符的路径。

- **前缀匹配**：当 `kind` 为 `prefix` 时，按前缀匹配。例如存在 `/api/user/app.js` 方法，开发者以 `prefix` 注册 `/api/user`，则前端路由 `/api/user/apps`、`/api/user/data` 都能匹配到同一个 handler；handler 内部可调用 `/api/user/app.js` 方法实现功能。
- **精准匹配**：需注册 `exact`。只有 `/api/user/app.js` 才能路由到该 handler，从而调用对应方法。

### handler：请求处理器

用法为 `handler(req, res)`，主要利用原生 `node:http` 的能力：

- 通过 `req.method` 判断请求方式（如 `GET` / `POST`）；
- 解析传入的参数，并通过 `res` 返回请求与响应；
- handler 内部应包含对传入参数的处理逻辑。

## fallback：兜底注册

注册方法为 **fallbackRegister**`(handler)`。由于所有**未匹配**的请求都会无条件到达这里，因此只需传入 `handler`。

fallback 具有**全局唯一性**：任何插件在注册时，只要该位置已被占用，注册一律失败。

> 由于开发内核架构要求，所有注册动作的返回值需要是disposer函数，用来保证时空可逆性
## 总结

本文档描述了 ct-host-web v2 的[[路由]]注册与兜底设计：通过 `hostRegister` 服务暴露 `kind` / `path` / `handler` 三参数接口，以[[前缀匹配]]与[[精准匹配]]两种模式分发请求；`fallbackRegister` 提供全局唯一的兜底处理器，接收所有未被路由匹配的请求。