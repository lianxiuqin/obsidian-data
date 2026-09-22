# ct-log-router

状态：🟡 初步设计（未开工）

> 现状：未开工。内核侧当前的自有诊断手段是启动层给 `ctx.logger` 挂的 stderr exporter（cordis 的 logger 默认只有内存环形缓冲、没有任何 sink）。
> 本插件落地后日志路由应能接管这件事，届时启动层那一段可以重新审视 —— 见 [[项目文档/项目设计文档/内核设计/启动层|启动层]] 的「装载期诊断」。
> 共性概念与通用对接约定见 [[项目文档/项目设计文档/插件设计/插件概念总览|插件概念总览]]。

**包名**：`@cordis-tavern/ct-log-router`（历史写法 `logRouter` 作废：包名与目录一律 kebab-case）

**功能**：日志分级与路由输出。

- **日志接口**：定义日志级别（Debug / Info / Warn / Error）与 Logger 接口，提供 `debug` / `info` / `warn` / `error` 方法与级别设置
- **日志路由**：注册 / 移除输出目标，按目标条件路由日志；`LogTransport`（name、minLevel、maxLevel、write）与 `LogEntry`（level、message、timestamp、source、data）**字段语义未定案**
- **输出目标**：`console`（格式化后输出到控制台）与 `file`（写入文件）
