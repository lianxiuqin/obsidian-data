# M4: 基础插件 - host

## 目标

实现 host 插件，提供 HTTP 服务和路由注册能力。

## 目录结构

```
ct-plugin-host/            # 独立插件包，不在 cordis-tavern 仓库内
├── package.json
├── cordis.patch.yml
└── src/
    ├── index.ts
    ├── server.ts
    └── router.ts
```

> 插件是 npm 包，开发完成后由 pnpm 安装到 `$CT_HOME/node_modules/`，供启动层按 `ct.profile` 清单加载。

## 核心功能

### server.ts

HTTP 服务模块，提供启动/停止服务、注册/移除路由功能。定义 RouteDefinition 接口（path、method、handler）和 RouteHandler 类型。

## 验收标准

- [ ] 可启动 HTTP 服务
- [ ] 其他插件可注册路由
- [ ] 路由可正常响应请求

---

> 上级文档：[[项目文档/开发文档/首期开发/首期开发总览|首期开发总览]]
