# ct-event-scope v2：有序监听 API

状态：🟢 已实现（`@cordis-tavern/ct-event-scope@0.2.0-alpha.1`；`on` / `emit` 有序监听）

> 架构不变：事件域 `register` / `injo`、独立总线、`emit` / `on`。
> 本页只解决一件事：**在 `on` 上表达「跑在谁前 / 谁后」**，并把排序规则写成契约。
> v1 见 [[ct-event-scope]]。

## 1. 目标

原版只能「监听 / 发布」，无法约束**同一事件**上多个监听器的执行顺序。
v2 在 `on` 上增加具名 `id` 与 `before` / `after` 约束，执行序由**拓扑**决定。

**不做**：不引入阶段枚举、不引入 pipeline / middleware、不改 emit 语义、不规定 id 命名格式。

## 2. API

### 2.1 `on`（唯一注册入口）

```ts
on(
  event: string,
  meta: ListenMeta,
  handler: Handler,
): Disposer
```

```ts
interface ListenMeta {
  /** 本监听器的全局唯一 id。字符串自由格式，契约不规定命名法。 */
  id: string
  /** 约束：排在这些 id 之前。可省。 */
  before?: string | string[]
  /** 约束：排在这些 id 之后。可省。 */
  after?: string | string[]
}

type Handler = (data: unknown, emit: EmitFn) => void | Promise<void>
type Disposer = () => void
```

| 项 | 约定 |
|---|---|
| `event` | 事件名，与 `emit(event, data)` 对称 |
| `meta.id` | **必填**。标识本监听器，供别人 `before` / `after` 引用 |
| `meta.before` / `after` | 值为**其它监听器的 id**（不是事件名）。单字符串或字符串数组 |
| 两侧都省 | **不参与排序**，与其它无约束监听器的相对顺序**不保证** |
| 返回值 | `disposer`，幂等；卸载时移除本监听器 |

> `id` 由开发者自定，**不强制命名约定**。冲突与拓扑问题一律显式报错（§4）。
> 后续以开发者工具辅助生成 / 检查 id，不在契约里加限制。

### 2.2 `emit`（发布）

```ts
emit(event: string, data: unknown): Promise<void>
```

- 按该事件**已解析的拓扑序**依次调用监听器。
- `handler` 可返回 `Promise`；`emit` **按序 await**（前一个结束才开始下一个），以兑现「严格顺序」。
- handler 抛错 / reject：记日志后**继续**后续监听器（与 v1 一致，不中断整条链）；不吞掉错误。

### 2.3 Handler 形参 `emit`

```ts
on('input', { id: 'prompt', before: 'llmApi' }, async (data, emit) => {
  const next = await enrich(data)
  emit('input', next)   // 加工后再发布，供后续监听器接收
})
```

- 第二参是**发布函数**，避免 handler 自己抓总线。
- 红线：在同一 `event` 上重发时，必须保证本监听器不会再次收到自己发出的那条（例如 `after` 更早的源、或保证链上不回环）。**自环在拓扑解析时抛错**（§4）。

## 3. 执行序规则

1. **拓扑为主**：以 `before` / `after` 边为准，求拓扑序。
2. **加载序 / 注册序不参与排序**：插件装载顺序与下载顺序绑定，**不稳定**，不得当作主序或次序。
3. **无约束节点**：既无 `before` 也无 `after` 的监听器，彼此之间**顺序不保证**；只要顺序有意义，必须写出约束。
4. **不可比的一对**：拓扑上无路径相连时，二者顺序不保证（不强行插入注册序打破平局）。

```
示例（event = 'input'）

  on('input', { id: 'llmApi' }, …)
  on('input', { id: 'prompt', before: 'llmApi' }, …)

  解析序：prompt → llmApi
```

## 4. 错误语义（一律显式报错，不静默）

| 情况 | 行为 |
|---|---|
| `meta.id` 缺失 / 空 / 非字符串 | 抛错，指名 event 与注册点 |
| **id 冲突**（同 event 下已有同 id） | 抛错，指名冲突双方（可带插件名） |
| `before` / `after` 指向不存在的 id | **前向引用允许**（加载序不确定）；`emit` 解析时仍悬空 → 抛错，指名悬空 id |
| **约束成环** | 抛错，打印环路径（如 `a → b → a`） |
| handler 抛错 / reject | 日志记录，继续执行后续监听器 |

> 冲突与成环**不**用「退化 / 忽略 / 按注册序」兜底 —— 那会把错误推迟到运行期且不可见。

## 5. 生命周期

- `disposer()`：移除本监听器及其出边约束；幂等。
- 域销毁 / 成员 `dis`：按 v1 语义清理，见 [[ct-event-scope]]。

## 6. 示例

```ts
// 发布
emit('input', payload)

// llmApi：最终调用方（相对位置靠后）
on('input', { id: 'llmApi' }, async (data) => {
  await callApi(data)
})

// prompt：加工后重发，且必须在 llmApi 之前
on('input', { id: 'prompt', before: 'llmApi' }, async (data, emit) => {
  emit('input', await buildPrompt(data))
})

// 无约束：不承诺与上面谁先谁后
on('input', { id: 'metrics' }, (data) => {
  record(data)
})
```

## 7. 明确不在本版

| 项 | 处置 |
|---|---|
| id 命名约定 / 强制前缀 | **禁止**（保持自由字符串；工具后续辅助） |
| `before()` / `after()` 快捷方法 | 本版不提供 |
| `order(event)` 序表查询 | 本版不提供；后续开发者工具兼容 |
| 阶段枚举 / pipeline | 不引入 |
| 以注册序 / 加载序打破平局 | **禁止**（来源不稳定） |

## 8. 验收（实现时）

1. `on` 仅接受 `on(event, meta, handler)` 形态；`meta.id` 必填。
2. `before` / `after` 支持 string 与 string[]。
3. 拓扑序正确；无约束监听器顺序不保证（测试不得断言其相对序）。
4. id 冲突 / 悬空引用 / 成环 → 抛错且信息含 id（环含路径）。
5. handler 返回 `Promise` 时按序 await；同步 handler 行为不变。
6. disposer 幂等；卸载后不再触发。
