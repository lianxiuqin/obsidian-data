---
tags:
  - 插件开发
  - slot
  - 前端渲染
---

# ct-ui-slots v2

状态：🟢 已实现（`list` + 组合式 `register` + `inject`；实现见 `cordis-tavern-plugin/ct-ui-slots/`）

v2 在 [[ct-ui-slots]]（v1）基础上做两件事：**扩展 `kind` 增加 `list` 类型**，并**调整 `register` 签名 + 新增 `inject` 方法**，让外部组件得以注册到具名插槽上。

## 新增插槽类型：`list`

`list` 是**可罗列型插槽**，用于**罗列多个不同的外部 [[React]] 组件**：

```ts
ctx.slots.register({ list: true }, name, component, children?)
```

| 项 | 说明 |
| --- | --- |
| 用途 | 在同一插槽下依次罗列多个外部组件 |
| 注册方 | 由**外部插件**主动注册（v1 的 `single` 为独占式，只保留权重最高者） |
| `kind` 声明 | **只声明 `list` 一项**——因为 `list` 属于外部注册型插槽，无需再叠加其他类型标记 |

> 与 `single` 的区别：`single` 是「同槽多声明、按 `priority` 取最高者胜出」；`list` 是「同槽多注册、**全部保留并依次罗列**」。

## 注册服务与新签名

外部注册统一走 **`slots` 服务**的 `ctx.slots.register()`，**组合式声明**（已实现正典）：

```ts
register({ name, id, order?, kind?, children? }, Component): disposer
```

| 字段 | 说明 |
| --- | --- |
| `name` | 分段式插槽名（见下） |
| `id` | 组件 id；同槽重复 → **告警**，保留 `order` 更小（靠前）者 |
| `order` | int，**小者靠前**；`list` 罗列次序，默认 `0` |
| `kind` | 缺省 `{ single: true, priority?: 0 }`；`list` 槽写 **`{ list: true }`**（只此一键） |
| `children` | 子槽声明 `{ 本地名: { kind, scope } }`；全名 = `父名.本地名` |

> 设计稿里同时出现过 `register({ list: true }, name, component, children?)` 旧式并列签名——**已废弃**，以本节组合式签名唯一为准。

消费方 `inject: ['slots']`；注册返回 `dis`（幂等）。

### `name`：插槽具名 + 分段式命名

`name` 是**插槽的具名标识**。由于 v2 引入外部注册，插槽数量与来源骤增，因此需要**具名检测**，防止具名冲突：采用**分段式命名法**——**同一作用域内、同层级的插槽不允许同名**。

示例：父插槽名为 `a`，子插槽名为 `b1`、`b2`；向 `b1` 插入内容时，使用的 `name` 是 **`a.b1`**。

### `id`：组件标识

`id` 是**组件 id**，用于**防止多个不同插件注册相同的功能性组件**：

- 发生**冲突时给出警告**；
- 冲突时依据 `order`，**显示靠前的组件**。

### `order`：排序依据

- 类型为 **int（整数）**，按大小排序，**数字越小越靠前**。
- 多组件共存于 `list` 插槽时，按 `order` 决定罗列次序。

## 新增方法：`inject`

```ts
ctx.slots.inject(key, callback)
```

| 参数 | 含义 |
| --- | --- |
| `key` | 等同于 `name`，即上例中的 **`a.b1`**——待注入的目标插槽 |
| `callback` | 条件满足时执行的动作，通常为 `register({ name, id, order }, Component)` |

`inject` 的意义是**条件式启用**：外部组件**等待目标插槽就绪后**再执行注入操作，避免插槽尚未声明时注册失败。

## 变更对照

| 维度 | v1 | v2 |
| --- | --- | --- |
| `kind` 类型 | 仅 `single` | 新增 `list`（`{ list: true }`） |
| `register` 签名 | `register(kind, name, component, children?)` | **`register({ name, id, order, kind?, children? }, Component)`** |
| 插槽命名 | 具名即可 | **分段式命名 + 同级同名检测**（`a.b1`） |
| 多组件共存 | 按 `priority` 取最高者 | `list` 全部保留，按 `order` 排序；`id` 冲突告警 |
| 条件注册 | 无 | 新增 `inject(key, callback)` |

## 总结

v2 是 [[ct-ui-slots]] 的迭代设计：核心是为 `kind` **新增 `list` 可罗列型插槽**，让多个外部 React 组件可被罗列到同一插槽上；为此把 `ctx.slots.register` 的签名从多参数并列改为 **`register({ name, id, order }, Component)`** 组合式声明，并借分段式命名、组件 `id` 冲突告警与 `order` 排序解决具名、冲突、顺序三个问题。最后新增 **`ctx.slots.inject(key, callback)`**，以条件式启用的方式等待插槽就绪后再完成注册注入。
