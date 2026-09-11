# M7: 基础插件 - logRouter

## 目标

实现 logRouter 插件，提供日志分级与路由输出。

## 目录结构

```
ct-plugin-log-router/      # 独立插件包，不在 cordis-tavern 仓库内
├── package.json
├── cordis.patch.yml
└── src/
    ├── index.ts
    ├── logger.ts
    ├── router.ts
    └── transports/
        ├── console.ts
        └── file.ts
```

> 插件是 npm 包，开发完成后由 pnpm 安装到 `$CT_HOME/node_modules/`。
> 包名采用 `ct-plugin-log-router`（kebab），与 `ct-plugin-host` 等保持一致；
> 目录名与文档名的 `logRouter` 为历史写法，待统一。

## 核心功能

### logger.ts

日志接口模块，定义日志级别（Debug、Info、Warn、Error）和 Logger 接口，提供 debug、info、warn、error 方法和级别设置功能。

### router.ts

日志路由模块，提供注册/移除输出目标、路由日志到匹配的输出目标功能。定义 LogTransport 接口（name、minLevel、maxLevel、write）和 LogEntry 接口（level、message、timestamp、source、data）。

### transports/

输出目标实现：
- console.ts：控制台输出，将日志格式化后输出到控制台
- file.ts：文件输出，将日志写入文件

## 验收标准

- [ ] 日志可按级别输出
- [ ] 支持控制台输出
- [ ] 支持文件输出
- [ ] 可动态添加/移除输出目标

---

> 上级文档：[[项目文档/开发文档/首期开发/首期开发总览|首期开发总览]]
