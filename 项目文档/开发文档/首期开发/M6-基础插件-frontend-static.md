# M6: 基础插件 - frontend-static

## 目标

实现 frontend-static 插件，提供静态资源托管和 fallback 路由。

## 目录结构

```
ct-plugin-frontend-static/ # 独立插件包，不在 cordis-tavern 仓库内
├── package.json
├── cordis.patch.yml
└── src/
    ├── index.ts
    ├── static.ts
    └── fallback.ts
```

> 插件是 npm 包，开发完成后由 pnpm 安装到 `$CT_HOME/node_modules/`。

## 核心功能

### static.ts

静态文件服务模块，提供设置资源目录、处理静态文件请求、获取 MIME 类型功能。

### fallback.ts

fallback 路由模块，注册为 host 的 fallback 路由，当 host 无法匹配时尝试处理请求。

## 验收标准

- [ ] 可访问静态前端页面
- [ ] host 无法匹配的路由能正确降级
- [ ] 支持常见的 MIME 类型

---

> 上级文档：[[项目文档/开发文档/首期开发/首期开发总览|首期开发总览]]
