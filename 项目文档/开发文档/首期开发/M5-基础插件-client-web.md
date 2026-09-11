# M5: 基础插件 - client-web

## 目标

实现 client-web 插件，负责前端声明发现、`window.boot` 排序和惰性加载控制。

## 目录结构

```
ct-plugin-client-web/      # 独立插件包，不在 cordis-tavern 仓库内
├── package.json
├── cordis.patch.yml
└── src/
    ├── index.ts
    ├── discovery.ts
    ├── boot.ts
    └── lazy.ts
```

> 插件是 npm 包，开发完成后由 pnpm 安装到 `$CT_HOME/node_modules/`。

## 核心功能

### discovery.ts

前端声明发现模块，扫描所有插件的 `ct.client` 声明，构建前端插件列表，按依赖关系排序。

### boot.ts

`window.boot` 管理模块，控制插件加载顺序，支持立即加载和延迟加载两种模式。

### lazy.ts

惰性加载控制模块，支持注册惰性加载模块、按需加载和预加载功能。

## 验收标准

- [ ] 可发现所有前端插件声明
- [ ] 可按依赖关系正确排序
- [ ] 惰性加载机制正常工作

---

> 上级文档：[[项目文档/开发文档/首期开发/首期开发总览|首期开发总览]]
