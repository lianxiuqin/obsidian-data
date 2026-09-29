# ct-host-web v3

状态：🟢 已实现（服务面收敛为 `webServer.register` / `fallbackRegister`；前缀段边界匹配；原生 handler）

> 承接 [[ct-host-web-v2|v2]]。本页为 **v3 正典**：统一服务 API、规范前缀匹配与 handler 生命周期。
> 实现以插件仓库 `src/index.ts` + `src/router.ts` 为准。

## 1. 服务面（唯一）

**只提供一个服务 `webServer`**，下辖两个函数方法。删除 `host` 服务及 `hostRegister` 旧名。

```ts
interface WebServerService {
  /** 注册路由；返回 disposer（幂等） */
  register(kind: 'prefix' | 'exact', path: string, handler: Handler): () => void
  /** 注册全局唯一 fallback；返回 disposer（幂等） */
  fallbackRegister(handler: Handler): () => void
}
```

| 旧（v2） | 新（v3） |
| --- | --- |
| `webServer.hostRegister` | **`webServer.register`** |
| `webServer.fallbackRegister` | 不变 |
| `host`（`{ name, server }`） | **删除**（监听细节不进服务面） |

消费方一律 `inject: ['webServer']`。

## 2. 路由参数规范

| 键 | 规则 |
| --- | --- |
| `kind` | 仅 `'exact'` \| `'prefix'` |
| `path` | 必须以 `/` 开头；**不去掉末尾 `/`** —— `/api` 与 `/api/` 是两条不同路由（尾斜杠是身份的一部分） |
| `handler` | 见 §3 |
| 身份 | `path` 全局唯一（与 kind 无关）；重复注册抛错 |
| 返回值 | **disposer**，幂等，只撤销自己那一条 / fallback 槽 |

### 2.1 exact

整串相等才命中（`pathname` 已剥 `?query`）。

### 2.2 prefix（段边界，规范）

注册 `prefix` + `P` 时命中条件：

```
pathname === P  或  pathname.startsWith(P) 且  pathname[P.length] === '/'
```

**即：前缀之后必须是段边界 `/`（或整串结束），不要求注册时以 `/` 结尾。**

| 注册 | 命中 | 不命中 |
| --- | --- | --- |
| `/api/user` | `/api/user`、`/api/user/`、`/api/user/data`、`/api/user/app.js` | `/api/userdata`、`/api/username`、`/api` |
| `/` | 一切以 `/` 开头的 pathname | — |
| `/api/` | `/api/` 自身（`pathname[5]` 起须为 `/` 或结束） | `/api`、`/api/x` |

多条 prefix 同时命中时，**最长 `path` 优先**；不引入注册顺序。

## 3. handler（原生，自管生命周期）

```ts
type Handler = (req: IncomingMessage, res: ServerResponse) => void
```

| 约定 | 说明 |
| --- | --- |
| **原生对象** | 不包装 `req` / `res`，不代理方法 |
| **响应权归 handler** | 命中后 router **不再**写状态码、不 `end`、不包 `try/catch` |
| **连接自管** | handler 决定何时 `end`，也可长期不 `end`（**SSE**、流式、WebSocket upgrade） |
| **升级 / 套接字** | 可用 `res.socket` / `req.socket` 做协议升级 |
| **异常** | handler 抛错不被 router 吞掉（由 Node 层处理）；建议 handler 自行捕获并响应 |

只有**未命中且无 fallback** 时 router 才写 `404` + `end`（避免连接悬挂）。

## 4. fallback

- `fallbackRegister(handler)`：全局唯一；占用后再注册 → 抛错
- 所有未命中请求无条件进入该 handler（同样自管生命周期）
- disposer 释放槽位后可再注册

## 5. 与 v2 的兼容说明

- **破坏性**：`hostRegister` 更名为 `register`；`host` 服务删除
- 监听地址 / 端口不再经服务暴露（测试与工具用 `startServer()` 导出或插件日志）

## 6. 配置

不变：`port` / `listen` / `ip` / `listenWhitelist`（见 [[ct-host-web|ct-host-web]]）。

## 7. 验收

1. 仅 `provide('webServer')`，无 `host`
2. `webServer.register` / `webServer.fallbackRegister` 为函数
3. `prefix /api/user` 命中 `/api/user/data`、不命中 `/api/userdata`
4. path 末尾 `/` 注册被规范化
5. handler 收到原生 `req`/`res`，可保持连接（SSE 形态测试）
