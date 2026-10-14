# Event Scope

> 为 Cordis 提供完全隔离的事件作用域。

## 简介

Event Scope 是 Cordis 的一个插件，在原生事件系统之外提供一套**完全隔离的事件总线**。每个作用域拥有独立的事件通道，内部信息不外传，外部信息无法传入。

以简单粗暴的隔绝方式，换取绝对的可靠性。

## 设计目标

- **完全隔离**：事件域之间互不可见，域内事件不冒泡到外部，外部事件不渗入域内。
- **无失败路径**：加入事件域不存在异步失败，只存在「已激活」与「等待激活」两种状态。
- **自动清理**：插件卸载时自动释放资源，无需手动干预。
- **零侵入**：不修改 Cordis 内核，作为普通插件提供服务。

## 边界限定

Event Scope **不是** Cordis 的原生 Context。它不承载服务、生命周期、依赖注入等任何原生能力，**仅用于事件的发布与监听**。

需要监听 Cordis 原生事件（如 `dispose`、`ready`）的插件，应继续使用自己的原始 `ctx`。

## 安装与启用

将本插件加入 Cordis 配置，启用后即可通过 `ctx.event` 访问服务。

## API

### `ctx.event.register(id: string)`

创建一个独立的事件域。

**参数**

- `id`：事件域的唯一标识符。

**行为**

- 若 `id` 未被占用，创建新的 Event Scope 并写入注册表。
- 触发状态机轮询，激活所有等待该 `id` 的 `injo` 请求。
- 通过 `ctx.effect` 注册自动清理。

**返回**

包装事件服务对象，包含 `on`、`emit`、`dis` 方法。

**幂等性**

返回对象的 `dis` 可被多次调用，第二次起为 no-op。

**错误**

若 `id` 已被其他插件注册，抛出错误并指明冲突双方：

```
error: id "auth" 已由插件 A 注册，插件 B 重复注册
```

---

### `ctx.event.injo(id: string): Promise<WrappedEvent>`

将当前插件加入指定事件域。

**参数**

- `id`：目标事件域的标识符。

**行为**

- 若 `id` 已注册，立即加入成员集合并 resolve。
- 若 `id` 未注册，进入等待队列，返回一个延迟 resolve 的 Promise。
- 通过 `ctx.effect` 注册自动清理。

**返回**

一个 Promise，resolve 后得到包装事件服务对象。

---

### 包装事件服务

`register` 与 `injo` 返回的对象结构一致，仅 `dis` 的语义不同。

| 方法 | 说明 |
| :--- | :--- |
| `on(event, listener)` | 在当前事件域内监听事件。 |
| `emit(event, ...args)` | 在当前事件域内发布事件。 |
| `dis()` | 释放当前持有的事件服务，语义见下表。 |

**`dis` 语义对比**

| 来源 | 行为 |
| :--- | :--- |
| `register` 返回对象 | 清退所有成员，清空等待队列，销毁事件域，从注册表删除，释放 `id`。 |
| `injo` 返回对象 | 仅从当前事件域中移除自己；若处于等待队列，则从队列中移除。 |

两者均由 `ctx.effect` 在插件卸载时自动调用，同时支持手动调用。

## 状态机

每个 `injo` 请求持有两种状态：

| 状态 | 含义 | 转换条件 |
| :--- | :--- | :--- |
| **waiting** | 事件域尚未注册，请求挂起 | `register(id)` 被调用时轮询激活 |
| **active** | 已加入事件域，可正常收发事件 | —— |

由于本插件以函数形式提供服务，不存在异步失败路径。`injo` 要么立即激活，要么进入等待，后续由 `register` 触发。**不会出现加入失败的情况。**

## 生命周期

```
插件 A: register('auth')
   ├─ 创建 EventScope，写入注册表
   ├─ 轮询等待队列，激活等待 'auth' 的请求
   └─ ctx.effect 注册自动 dis

插件 B: injo('auth') → Promise
   ├─ auth 已存在  → 立即 resolve，加入成员
   ├─ auth 不存在  → 进入等待队列，返回 pending Promise
   └─ ctx.effect 注册自动 dis

插件 A 卸载  →  dis 触发  →  整个事件域销毁
插件 B 卸载  →  dis 触发  →  B 从成员或等待队列中移除
```

## 数据结构

```ts
type EventMap = Map<string, EventScope>

interface EventScope {
  id: string
  emitter: EventEmitter        // 独立事件总线
  members: Set<Fiber>          // 已激活的成员
  waiting: Set<WaitingEntry>   // 等待激活的请求
}

interface WaitingEntry {
  fiber: Fiber
  resolve: (svc: WrappedEvent) => void
}
```

## 错误处理

### 注册 id 冲突

抛出错误，信息包含冲突 id 与双方插件名：

```
error: id "auth" 已由插件 A 注册，插件 B 重复注册
```

### injo 等待超时

采用**静默等待**策略。因为其他插件的注册耗时不可知，`waiting` 条目可能长期挂起。

- 输出 `warn` 级别日志，记录仍在等待的 `id`。
- 提供可选的超时配置项，超时后显式报错。
- 提供查询接口 `ctx.event.pending()`，返回当前所有等待中的请求，供调试使用。

### 等待条目的清理

不主动清理等待队列。插件卸载时，其 `ctx.effect` 会触发 `dis`，自动将自身从等待队列中移除，因此不会因插件卸载而残留。

若插件仍在运行但对应 `register` 始终未发生，等待条目会一直存在，直到插件自身卸载或 `register` 最终被调用。

## 设计约束

- **监听器顺序不保证**。如需确定性顺序，请单独引入 `dsh-plugin-hooks-ordering`。
- **不监听 Cordis 原生事件**。包装事件服务仅作用于本事件域内。
- **不承载原生能力**。服务、生命周期、依赖注入请使用原始 `ctx`。

## 使用示例

```ts
// 插件 A：创建事件域
const auth = ctx.event.register('auth')

auth.on('login', user => {
  console.log('login:', user)
})

auth.emit('login', { name: 'alice' })


// 插件 B：加入事件域
const authB = await ctx.event.injo('auth')

authB.on('login', user => {
  console.log('B received:', user)
})
```

## 设计理念

Cordis 内核只做三件事：管理插件生命周期、解析依赖、传播上下文。高级能力由插件生态补全。

Event Scope 遵循这一哲学，以极简机制实现事件隔离：

- **register 创建 + injo 加入**：一对核心 API，职责分明。
- **状态机轮询**：先到先等，register 触发激活，无异步失败。
- **effect 自清理**：依赖 Cordis 的 fiber 机制，卸载安全可靠。
- **Promise 返回**：以 `await` 等待加入，写法自然。