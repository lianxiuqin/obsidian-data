---
tags:
  - 插件开发
  - slot
  - 前端渲染
---

# ct-ui-slots

状态：🟡 设计中（未开工；`slot` 服务面已定 `register()` / `renderSlot(name, owner)`，插槽类型 v1 仅 `single`，`root` 为预设占位插槽）

## 目标

实现 **slot（插槽）** 的申请与使用：各 UI 插件在**自己的页面**上申请 slot，由本插件统一登记与管理，再交给渲染插件 `ct-ui-renderer` 精准接入内容。所有插槽最终统一挂在 **`root` 根插槽**下。

## 服务面

通过 [[cordis]] 的服务机制暴露 slot 内的函数。服务侧调用会返回一个 **`dis` 方法**，用于**释放副作用**——这是 cordis 的设计契约要求（见 [[插件契约层]]）。

消费方以 `inject: ['slot']` 拿到 `ctx.slot` 后使用。

## 函数

### 注册函数：`register()`

申请一个 slot；除自身类型外，还可**声明子插槽**。

```ts
ctx.slot.register(kind, name, component, children?)
```

| 参数 | 含义 |
| --- | --- |
| `kind` | **插槽类型**。v1 仅提供 `single`，用于声明**独占式插槽**；且大多以**组合型声明**的形式给出（`{ ... }` 对象），而非裸字符串 |
| `name` | **插槽名称**。受插件内核限制，**所有插槽必须具名** |
| `component` | **UI 内容**，即该插件自身的 [[React]] 组件 |
| `children?` | **子插槽声明**，可选。用于在当前插槽下再挂一层插槽 |

### 子插槽声明：`children?`

子插槽的声明方式与**主插槽**基本一致，只是**嵌套在 `children` 内部**完成。基础结构为「插槽名 → `{ kind, scope }`」：

```ts
children: {
  'name': { kind, scope },
}
```

| 键 | 含义 |
| --- | --- |
| `'name'` | **子插槽名称**，同样必须具名 |
| `kind` | 子插槽的**类型**，取值同主插槽 |
| `scope` | **上下文归属**，一般归属 `root` |

示例：

```ts
children: {
  'sidebar': { kind: 'single', scope: 'root' },
  'main': { kind: 'keyed', scope: 'root' },
}
```

> 示例中的 `keyed` 属于非 `single` 类型，本文 v1 类型表尚未定义它，多组件共存规则待补。

### 渲染函数：`renderSlot()`

**唯一入口，签名恒为两参**——服务侧与组件侧是**同一个函数**，不因调用位置而改名。

```ts
ctx.slot.renderSlot(name, owner)
```

| 参数 | 含义 |
| --- | --- |
| `name` | **槽位键名**，作为 key 识别目标插槽（受内核限制必须具名） |
| `owner` | **目标组件所需的参数**，即最终传给它的 **props**。**两参恒定**：该插槽不需要参数时传空对象 `{}`，**不可省略** |

> ⚠ **订正**：早期草稿写作 `renderer(name, component)`，其中 `component` 是笔误，实为 **`owner`**；另一处示例漏写后参、写成 `renderSlot('sidebar')`，同样更正为两参调用。

两种调用位置：

| 位置 | 调用 | 作用 | 返回 |
| --- | --- | --- | --- |
| **服务侧** | `ctx.slot.renderSlot(name, owner)` | 顶层**开发前期固定 `name = 'root'`**，向该槽位提交 `owner` | `dis` |
| **组件侧** | `renderSlot('child', owner)` | 在父插槽组件内渲染指定子插槽 | 渲染结果 |

**递归分发规则**：

- **props 结构恒定**：槽位组件拿到的 **props = 调用方传入的 `owner` + 系统注入的 `renderSlot`**。因此**任意层级**的组件都能拿到 `renderSlot`，**递归链连续**，嵌套多少层写法都不变。
- **最终挂载点不变**：无论递归多少层，组件最终仍挂在 **`root`** 上。

### 父插槽的渲染写法

```jsx
function Parent({ sessionId, renderSlot }) {
  return <div>{renderSlot('child', { sessionId })}</div>
}
```

| 片段 | 含义 |
| --- | --- |
| `Parent` | 父插槽的 [[React]] 组件名（一般即插件自身的组件名） |
| `{ sessionId, renderSlot }` | **props 整体**——两者都是 prop，只是组合成 React 的 props 出现 |
| `sessionId` | 该 UI 的**必要参数**，出现在 `owner` 里 |
| `renderSlot` | **渲染器函数**，由系统注入，属于 props 的一部分 |
| `{ sessionId }` | 传给 `child` 子插槽的 `owner` |

> **`root` 在示例中没有参数，不代表 props 为空**：即便 `owner` 是 `{}`，组件 props 里依然有系统注入的 `renderSlot`。

## 插槽类型：`single`

`single` 是**独占式插槽声明**，组合型声明形如 `{ single, priority? }`：

- `priority` 未声明时，**默认值为 `0`**。
- 当**同一插槽有多个组件声明**时，按 **`priority` 权重最大值**渲染，即只有优先级最高的那个组件胜出。
- 该规则对 **`root`** 同样成立：多个插件以同名 `root` 声明时，按权重决出占位者。

## 特殊设置：`root`

`root` 是插件在**自建时**申请出来的**预设占位插槽**——它**自带一个空的占位实现**，存在的意义就是**被占位**。

- **承担渲染整个 UI 的职责**，同时是**挂载所有插槽的根插槽**：所有子插槽的内容最终都收束到 `root` 上。
- **原生为空实现**：若在没有 [[React]] 组件的情况下被调用，由于是空实现，会直接导致系统级报错，**没有 fallback（兜底）**。
- **属性为 `single`，`priority` 未声明**（即 `priority = 0`）：这一条正是给其他主页面结构组件**预留的占位入口**。
- **`owner` 由占位者提供**：`root` 自身并不决定 `owner`，而是由**注册了同名 `root`、但权重不同的其他插件**提供。谁的 `priority` 最高，谁就**同时接管 `root` 的渲染组件与 `owner`**（以 `renderSlot('root', owner)` 提交），并把 props 上的 `renderSlot` 向下递归分发。

## 总结

本文档定义 `ct-ui-slots` 插件的**目标**与接口：通过 [[cordis]] 服务暴露 `register()`（申请 slot，用 `children` 嵌套声明子插槽、`scope` 标明上下文归属）与 **`renderSlot(name, owner)`**——唯一的两参函数，`name` 为槽位键名、`owner` 为目标组件所需的 props，服务侧提交、组件侧渲染，同名保证递归链连续，组件最终仍挂载于 `root`。**`root`** 是自建时申请、**自带空实现且无兜底**的**预设占位插槽**，其渲染组件与 `owner` 一并由**同名注册、权重最高**的其他插件接管。槽位组件的 props 恒为「`owner` + 注入的 `renderSlot`」，故 **`root` 即使无参数也不等于 props 为空**。v1 只提供 **`single`** 一种独占式插槽类型，同槽多组件时按 **`priority`** 取最大值渲染。
