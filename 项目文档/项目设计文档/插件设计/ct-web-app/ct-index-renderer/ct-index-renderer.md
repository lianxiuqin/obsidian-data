# ct-index-renderer

状态：🟢 已实现（源码在 `cordis-tavern-plugin/ct-index-renderer/`；载荷含 `placement: 'global'`）

> 前言：主页面渲染需要 index，而 ct-client-found 承载服务过多——本插件把 **index 插入**与 client-found 解耦，在**启动时段**收集片段并统一写入 index。
> 发布方示例：ct-client-found 按固定序 `emit` 四条（见 [[ct-client-found-v2]]）。本页是事件线正典。

## 职责

| 项 | 说明 |
| --- | --- |
| 事件 | 固定名 **`index-inject`**（`INDEX_INJECT_EVENT` / 导出 `event` 同值） |
| 发布 | `ctx.emit('index-inject', payload)` |
| 订阅 | 本插件 apply 时 `ctx.on('index-inject', …)` → `indexInject` 入表 |
| 插入 | `apply(html)`：按 placement 写入模板 |
| 服务 | `ctx.provide('indexRenderer', { indexInject, entries, apply, clear })` |
| 自有路径 | 另 provide **`distIndex`**：`ctx.distIndex()` → **本包** `dist/index.html` 绝对路径（函数形态；与 client-found 无关） |

 Cordis 事件原语：`ctx.emit` / `ctx.on`（订阅方也可用服务方法 `indexInject` 走同一条校验）。

## 载荷契约

```ts
{ kind: string, placement: 'head' | 'body' | 'global', content: string }
```

| 键 | 类型 | 规则 |
| --- | --- | --- |
| `kind` | string | 非空；类型提示（`script` / `inline` / `global` / `style` …），不强制枚举 |
| `placement` | `'head' \| 'body' \| 'global'` | 仅这三值；非法 → `TypeError` 指名值 |
| `content` | string | 非空；浏览器 DOM 片段**原文**（或 `global` 赋值脚本），原样插入不包标签 |

未知键 → `TypeError`。校验失败 **不入表**。

## 插入规则（`apply`）

| placement | 落点 |
| --- | --- |
| `global` | **文档最前**（前缀到 HTML 开头；无 head/body 位置约束，须先于 head 脚本执行，如 `__CT_BOOT__`） |
| `head` | 有 `</head>` → 插其前；否则整段前缀 |
| `body` | 有 `</body>` → 插其前；否则整段追加末尾 |

- **缺闭合标签不丢注入**（前缀 / 追加兜底）。
- **同一 placement 内保持收集序（发布序）**。
- **发布序 ≠ 执行序**：执行顺序由 apply 后的文档位置决定（`global` 最先，再 head，再 body）。

## 服务面

```ts
export interface IndexRendererService {
  indexInject(raw: unknown): IndexInjectPayload  // 与事件订阅同一入口/校验
  entries(): readonly IndexInjectPayload[]
  apply(html: string): string
  clear(): void
}
```

消费方：`export const inject = ['indexRenderer'] as const` → `ctx.indexRenderer.apply(template)`。

## 配置

无业务键：`undefined` / `null` / `{}` 通过；任意未知键 → `TypeError`。

## 装载

- 包名：`ct plugin add -p <环境名> @cordis-tavern/ct-index-renderer`
- 目录：`$CT_HOME/profile/<env>/node_modules/` 或 `$CT_HOME/node_modules/`
- `ct.bundle.path` → `cordis.patch.yml`，`insert[].id: indexRenderer`
