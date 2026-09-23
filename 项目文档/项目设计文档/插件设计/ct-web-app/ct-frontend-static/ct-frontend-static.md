# ct-frontend-static

状态：🟡 初步设计（未开工）

> 现状：未开工。共性概念与通用对接约定见 [[项目文档/项目设计文档/插件设计/插件概念总览|插件概念总览]]。

**包名**：`@cordis-tavern/ct-frontend-static`

**功能**：静态资源托管与 fallback 路由。

- **静态文件服务**：设置资源目录、处理静态文件请求、给出 MIME 类型
- **fallback 路由**：注册为 host 的 fallback 路由，host 匹配不到时交它处理

**依赖**：用 `inject` 声明对 host 服务的依赖；host 未就绪时本插件保持 PENDING（见 [[ct-host-web|ct-host-web]]）。

开发
限制，由于运行时需要一张构建产物与路由的对应表，所以此项目不能主动启动，反而要依赖ct-client-web,插件对表进行总览后，提取所有构建产物和路由关系，总结为一张对应表后，通过cordis框架的原生工具ctx.plugin()进行启动，ctx.plugin(plugin,config)参数中，config可以通过cordis.include插件提供的！！js解析动态文件。