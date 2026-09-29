# ct-log-router

状态：🟢 已实现（`0.0.1-alpha.2`，插件仓库 `cordis-tavern-plugin/ct-log-router/`）

> 现状：已交付。内核侧当前的自有诊断手段是启动层给 `ctx.logger` 挂的 stderr exporter（cordis 的 logger 默认只有内存环形缓冲、没有任何 sink）。
> 本插件落地后日志路由可接管更多输出目标；启动层 stderr 段可再审视 —— 见 [[项目文档/项目设计文档/内核设计/启动层|启动层]] 的「装载期诊断」。
> 共性概念与通用对接约定见 [[项目文档/项目设计文档/插件设计/插件概念总览|插件概念总览]]。
> **输出两路（已实现）**：① 终端 `process.stdout`；② `persist.json` → `$CT_HOME/data/logger/<yyyy-mm-dd>.json`（namespace=`logger`，persist 的 `data/` 固定前缀；`has` 后追加/新建）。`inject: ['persist']`。
> **级别语义（已按 vendor/cordis 校准）**：`error=0 / warn=1 / info=2 / debug=3`；`levels` 阈值表示「`level <= threshold` 才输出」。`default: 3` 全收，`default: 0` 只收 error。

## 简介

本插件为 Cordis 应用提供一个自定义日志导出器（Exporter），用于接管或扩展 `ctx.logger` 的输出行为。通过注册自定义 Exporter，你可以将日志写入文件、发送到远程服务、按自定义格式输出到控制台，或同时分发到多个目标。

Cordis 的 `ctx.logger` 本身只负责收集和缓冲日志，真正的输出由注册在其上的 Exporter 决定。本插件不替换 `ctx.logger` 服务，而是为其添加一个可配置的导出端。

---

## 设计目标

- **可插拔**：通过 Cordis 插件机制注册 Exporter，不修改核心服务。
- **多目标支持**：可同时注册多个 Exporter，日志会分发到所有注册者。
- **级别可控**：可配置接收哪些级别的日志，避免丢失 `warn`、`debug` 等信息。
- **生命周期安全**：使用 `ctx.effect()` 管理资源，插件卸载时自动清理。
- **格式化灵活**：支持自定义日志格式化函数。

---

## 安装与注册

在 `cordis.yml` 中注册插件：

```yaml
- name: './logger-custom-exporter.ts'
  config:
    levels:
      default: 3        # 0=error … 3=debug；3=全收
    format: 'json'      # 输出格式：json / text / 自定义
    target: 'file'      # 输出目标：console / file / remote
    filePath: './logs/app.log'
```

注册后，`ctx.logger` 产生的日志会同时进入默认缓冲区和本 Exporter。

---

## 配置项

| 配置项 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `levels` | `object` | `{ default: 0 }` | 各命名空间的日志级别阈值 |
| `format` | `'json' \| 'text' \| Function` | `'text'` | 日志格式化方式 |
| `target` | `'console' \| 'file' \| 'remote'` | `'console'` | 输出目标 |
| `filePath` | `string` | `'./logs/app.log'` | `target: file` 时的文件路径 |
| `remoteUrl` | `string` | — | `target: remote` 时的接收地址 |
| `includeName` | `boolean` | `true` | 输出是否包含日志来源插件名 |
| `includeTimestamp` | `boolean` | `true` | 输出是否包含时间戳 |

---

## 日志级别说明

Cordis 日志级别（`LoggerLevel`，与 `vendor/cordis` 一致）：

| 方法 | 数值 | 说明 |
|---|---|---|
| `error` | 0 | 错误 |
| `warn` | 1 | 警告 |
| `info` | 2 | 常规信息 |
| `debug` | 3 | 调试 |

过滤条件是 **`level <= threshold`** 才交给 Exporter（`levels[name] ?? levels.default ?? INFO`）。

| 阈值 | 收到 |
|---|---|
| `default: 3` | 全部（本插件默认） |
| `default: 2` | error / warn / info |
| `default: 0` | 仅 error |

> 旧稿写「`default: 0` 接收所有级别」与「error=0 / info=1 / warn=2」均与实现不符，已订正。

---

## 使用示例

### 输出到控制台（文本格式）

```yaml
config:
  levels: { default: 3 }
  format: text
  target: console
```

### 输出到文件（JSON 行格式）

```yaml
config:
  levels: { default: 3 }
  format: json
  target: file
  filePath: './logs/app.log'
```

### 同时输出到控制台和文件

可以注册两个 Exporter 实例，或在本插件内部分发到多个目标。

---

## 自定义 Exporter 开发指南

若需编写自己的 Exporter 插件，遵循以下步骤：

### 1. 声明插件元信息

```ts
export const name = 'logger-custom-exporter'
export const inject = ['logger']
```

### 2. 在 `apply` 中注册 Exporter

```ts
export function apply(ctx, config) {
  ctx.effect(() => ctx.logger.exporter({
    handle: (record) => {
      // record: { level, message, name, timestamp, ... }
      const output = formatRecord(record, config)
      writeToTarget(output, config)
    },
    levels: config.levels ?? { default: 3 }
  }))
}
```

`ctx.logger.exporter()` 返回一个清理函数，`ctx.effect()` 确保插件卸载时自动调用，释放文件句柄、网络连接等资源。

### 3. 处理日志记录

`handle` 接收的 `record` 对象通常包含：

| 字段 | 类型 | 说明 |
|---|---|---|
| `level` | `number` | 日志级别 |
| `message` | `string` | 日志内容 |
| `name` | `string` | 来源插件名 |
| `timestamp` | `number` | 时间戳 |
| `args` | `any[]` | 原始参数 |

### 4. 格式化与输出

可根据 `config.format` 选择：

- `text`：`[时间] [级别] [来源] 消息`
- `json`：`JSON.stringify(record)`
- 自定义函数：`(record) => string`

输出目标同理，支持控制台、文件流、HTTP 请求等。

---

## 生命周期与资源清理

- 所有 Exporter 必须通过 `ctx.effect()` 注册，确保卸载时清理。
- 若使用文件流，清理函数中应 `stream.end()`。
- 若使用网络连接，清理函数中应关闭连接或取消订阅。
- 异步清理函数会被等待，但多个清理函数之间不保证顺序，有顺序依赖的清理应放在同一个函数中。

---

## 最佳实践

- **需要全量日志时用 `levels.default: 3`**（0 只收 error）。
- **避免在 `handle` 中执行阻塞操作**，如需写文件，使用异步 API。
- **格式化函数应轻量**，避免影响日志性能。
- **生产环境建议同时保留默认缓冲 Exporter**，便于调试。
- **多 Exporter 场景注意重复输出**，可通过级别或命名空间过滤。

---

## 扩展方向

- 支持日志轮转（按大小或日期切割）。
- 支持远程日志聚合（如 Loki、Elasticsearch）。
- 支持彩色控制台输出。
- 支持按插件名动态调整级别。
- 支持采样和限流。

---

## 示例项目结构

```text
cordis-plugin-logger-custom-exporter/
├── src/
│   ├── index.ts          # 插件入口与 apply
│   ├── formatter.ts      # 格式化逻辑
│   └── targets/
│       ├── console.ts
│       ├── file.ts
│       └── remote.ts
├── README.md
└── package.json
```

---

以上内容可作为该插件的 `README.md` 初稿。核心思想是：**通过 `ctx.logger.exporter()` 注册自定义导出逻辑，用 `ctx.effect()` 管理生命周期，并显式设置级别阈值以确保日志不丢失。**