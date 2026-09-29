---
tags:
  - 插件开发
  - slot
  - 前端渲染
---

# ct-ui-slots

状态：🟢 已实现（`0.1.0-rc.1`；服务面 `register()` / `renderSlot(name, owner)`，v1 仅 `single`，`root` 为预设占位插槽）

## 目标

实现 **slot（插槽）** 的申请与使用：各 UI 插件在**自己的页面**上申请 slot，由本插件统一登记与管理，再交给渲染插件 [[ct-ui-renderer]] 精准接入内容。所有插槽最终统一挂在 **`root` 根插槽**下。

## 运行形态

- **服务只存在于浏览器端的 Cordis 实例**：由 [[ct-client-cordis]] 装载进浏览器实例（完整 cordis 打包进浏览器），与其他 UI 插件同实例。
- **Node 端保留空 apply 并被真装载**——否则进不了 [[ct-client-found]] 的发现/下载链路；空 apply 不 provide 任何服务。
- 包形态走 [[插件契约层]]：双入口（`exports["."]` / `exports["./client"]`），`ct.bundle` 的 insert id 为字符串 **`'slots'`**，`ct.client` 声明 `platform: web`。
- client bundle 经 `__CT_MODULES__` 工厂表登记；factory **必须**返回 `{ name, apply }`（可含 `inject` / `provide` 等元数据）。

## 服务面

服务名 **`slots`**（与 insert id 一致）。消费方以 `inject: ['slots']` 拿到 `ctx.slots` 后使用。服务侧调用的注册函数返回 **`dis`** 方法，用于释放副作用——这是 cordis 的设计契约要求（见 [[插件契约层]]）。

插件间依赖一律用**具体 Cordis 服务名**的 `inject`，不用时机钩子。

## 函数

### 注册函数：`register()`

**`register()` 是浏览器侧 Cordis 服务 `ctx.slots` 的方法，只存在于浏览器端的 Cordis 实例中，返回 `dis`。** 申请一个 slot；除自身类型外，还可**声明子插槽**。

```ts
ctx.slots.register(kind, name, component, children?)
```

| 参数          | 含义                                                                         |
| ----------- | -------------------------------------------------------------------------- |
| `kind`      | **插槽类型**。v1 仅提供 `single`，组合型声明形如 `{ single: true, priority? }`，而非裸字符串      |
| `name`      | **插槽名称**。受插件内核限制，**所有插槽必须具名**                                              |
| `component` | **UI 内容**，即该插件自身的 [[React]] 组件——JS 函数对象，在同一个浏览器 JS 上下文中被引用和存储（不过端传输、无标识映射） |
| `children?` | **子插槽声明**，可选。用于在当前插槽下再挂一层插槽                                                |

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
| `scope` | **上下文归属**，一般归属 `root`（v1 仅记录，不参与解析） |

示例：

```ts
children: {
  'sidebar': { kind: { single: true }, scope: 'root' },
}
```

### 渲染函数：`renderSlot()`

**`renderSlot()` 是纯渲染函数，不返回 `dis`**（副作用释放只属于 `register`）；在 React 组件树内调用时返回**渲染结果**。唯一入口，签名恒为两参。

```ts
ctx.slots.renderSlot(name, owner)
```

| 参数 | 含义 |
| --- | --- |
| `name` | **槽位键名**，作为 key 识别目标插槽（受内核限制必须具名） |
| `owner` | **目标组件所需的参数**，即最终传给它的 **props**。**两参恒定**：该插槽不需要参数时传空对象 `{}`，**不可省略** |

两种调用位置：

| 位置 | 调用 | 作用 | 返回 |
| --- | --- | --- | --- |
| **渲染顶层** | `ctx.slots.renderSlot('root', owner)` | 向根插槽渲染（开发前期固定 `name = 'root'`） | 渲染结果 |
| **组件内** | `renderSlot('child', owner)` | 在父插槽组件内渲染指定子插槽 | 渲染结果 |

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

### props / owner 的两种来源

1. **声明式**：声明了 `children` 的插槽会收到 `renderSlot` prop 用来渲染子插槽，业务数据自然由 owner 提供——这类主要提供工具函数和业务级数据。更详细的定义与调用方法后续添加，**v1 只开发基础功能**。
2. **系统上下文与父级上下文**：系统上下文未来会有多种结构性变量（如 `runTime` 运行时长等基础变量），会挂载到 cordis 的 root 根节点供所有插件使用；对子插槽 UI，父插槽调用 `renderSlot(key, owner)` 实际上是把 owner 传给子插槽使用。

> **owner 语义**：owner 不是单纯的跨端数据，而是复杂的运行时数据，通过访问不同的作用域来实现定义，甚至需要访问其他业务来实现。这套机制因复杂性太高 **v1 未实现**——v1 中 owner 为调用方当场传入的对象，原样进 props。

## 插槽类型：`single`

`single` 是**独占式插槽声明**，组合型声明形如 `{ single: true, priority? }`：

- `priority` 未声明时，**默认值为 `0`**。
- 当**同一插槽有多个组件声明**时，按 **`priority` 权重最大值**渲染，即只有优先级最高的那个组件胜出；平局取后登记者。
- 该规则对 **`root`** 同样成立：多个插件以同名 `root` 声明时，按权重决出占位者。
- `register` 返回的 `dis` 幂等；撤销后按剩余登记重新决出。

## 特殊设置：`root`

`root` 是插件在**自建时**申请出来的**预设占位插槽**——它**自带一个空的占位实现**，存在的意义就是**被占位**。

- **承担渲染整个 UI 的职责**，同时是**挂载所有插槽的根插槽**：所有子插槽的内容最终都收束到 `root` 上。
- **原生为空实现**：若在没有被接管的情况下被调用渲染，直接导致系统级报错，**没有 fallback（兜底）**。
- **属性为 `single`，`priority` 未声明**（即 `priority = 0`）：这一条正是给其他主页面结构组件**预留的占位入口**。
- **接管**：注册了同名 `root` 且权重更高的插件接管**渲染组件**；owner 不被登记携带，始终由每次 `renderSlot` 调用当场传入。

## 总结

`ct-ui-slots` 在**浏览器端 Cordis 实例**（[[ct-client-cordis]] 装载）中提供服务 **`slots`**：`register()` 是该服务的方法，返回 `dis` 释放副作用；`renderSlot(name, owner)` 是纯渲染函数（不返回 dis，组件树内返回渲染结果），`owner` 为目标组件 props、两参恒定。槽位组件的 props 恒为「`owner` + 注入的 `renderSlot`」，递归链连续，组件最终挂载于 `root`。`root` 是自建时申请、**自带空实现且无兜底**的预设占位插槽，其渲染组件由同名注册、权重最高者接管。v1 只提供 **`single`** 独占式插槽类型，同槽多组件按 `priority` 取最大值渲染；owner 的作用域解析机制 v1 不实现。
