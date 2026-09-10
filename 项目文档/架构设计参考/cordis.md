# corids架构阐述文档
## 一、Cordis 是什么

Cordis（拉丁语意为“心”）是一个元框架（Meta Framework）—— 即“用于构建框架的框架”。它不耦合任何具体业务领域，只专注于解决一个核心问题：如何让软件的各个组件可以安全地组合、解绑与重组。Cordis 以 vendor 方式被引入 DeepSeek Harness（dsh），作为其底层插件框架。在 dsh 中，“一切皆插件”是核心架构宣言：模型适配器、工具注册表、会话日志、Agent 智能体循环本身，全部是插件。每项能力——包括工具、LLM 适配器、文件访问乃至 agent loop——都是挂载到共享上下文中的插件。

Cordis 的设计已被形式化为一篇论文《A Programming Paradigm for Spatiotemporal Composability》（时空可组合性的编程范式）。其与传统依赖注入框架的本质区别在于：传统 DI 假设“一旦绑定，服务就一直在”；Cordis 的假设是“服务可以随时出现，也可以随时消失”。提供方被卸载时，所有依赖它的插件自动卸载（effect 回滚）；新提供方就绪后，自动重载。依赖方不需要写任何重连代码。

Cordis 上游仓库为 `cordiverse/cordis`，在 Koishi 生态中经历了上千插件的并发运行考验，注册与注销的边界问题已被长期实战覆盖。Cordis 的内核极薄——核心逻辑约 2000 行 TypeScript，运行时零依赖，只依赖 Node 和 TypeScript 的内置能力。它的设计哲学极其专注，只做三件事：**生命周期管理**（插件的加载与卸载）、**插件解析**（处理插件间的依赖关系）、**上下文传播**（管理插件间的数据共享）。除此之外，路由、命令、网络、存储、权限，统统交给插件。


## 二、五个核心概念

Cordis 的全部语义可以浓缩为五个概念，由浅入深逐一展开：

**插件是实现 Service 的对象。** 它可以是一个带有可选 `inject` 和 `apply(ctx)` 字段的函数，也可以是一个 Service 子类，其生命周期由 Cordis 挂载到当前上下文中。

**上下文是服务的容器。** 一个服务占据一个稳定的 `ctx.<key>`，其他插件通过 key 查找服务，而非导入具体实现。

**通过 inject 声明服务依赖。** 插件声明所需的服务后，会等待这些服务就绪才启动；加载顺序通过服务依赖表达，而非手动编排启动序列。

**类型化事件用于通信。** 服务通过 TypeScript 声明合并注册事件名，然后以 emit、waterfall、parallel、serial 或 bail 方式分发，分别对应监听者观察、包装、并行扇出、按序执行或停在首个 bail 值。

**注册是可逆的副作用。** 提示词片段、工具 schema、适配器、提供方和监听器通过 `ctx.effect()` 或 `ctx.on()` 安装，reload 和 teardown 时会按预期撤销。


## 三、插件（Plugin）

插件是 Cordis 中的基本执行单元。Cordis 支持三种插件入口形态：

- **函数插件**：以 `(ctx, config)` 为参数调用
- **类插件**：以 `(ctx, config)` 构造，需继承 `Service` 或实现相应接口
- **对象插件**：具有 `apply(ctx, config)` 方法

所有插件形态共享以下元数据：

- `name`：用于 fiber 诊断和日志名称的显示名称
- `Config`：在插件启动前应用于配置的 Standard Schema 验证器
- `inject`：插件所需的服务；仅在所有服务可用时才加载
- `provide`：插件提供的服务名称
- `intercept`：插件声明消费其拦截配置的服务名称

Cordis 本身接受任意 Standard Schema 验证器，因此将普通对象导出为 `Config` 无法工作。错误配置会导致加载失败，并给出准确的错误——插件绝不会在配置不完整时启动。

**嵌套上下文。** `ctx.plugin()` 创建一个子 Fiber，它继承父上下文但拥有独立的生命周期。子插件随父插件一同卸载。


## 四、上下文（Context）

Context 是 Cordis 的核心对象：每个服务、事件和生命周期 API 都通过 `ctx` 访问。Context 是一个代理：正常的属性读取通过服务解析器进行，而 `extend()`、`isolate()` 和 `intercept()` 创建作用域子上下文而不改变其父级。

**根上下文与子上下文。** Root Context 是 Cordis 的顶级依赖容器。`new Context()` 创建根依赖容器。Context 提供三种作用域化操作：

- **`ctx.extend(meta?)`**：创建在当前作用域之上附加额外元数据的子上下文。子上下文原型继承父级的所有属性；`meta` 的自有属性遮蔽继承的属性。父级不被修改。
- **`ctx.isolate(name, label?)`**：为 `name` 创建具有独立服务作用域的子上下文。在返回的上下文下方，`name` 服务的读写针对新标签解析，而不是父级的，因此可以提供不同的实现而不影响父作用域。将相同的 `label` 传递给两次 `isolate()` 调用会合并它们的作用域。只有包含在 `keys` 中的服务会在新上下文中被隔离，未包含的服务仍与父上下文共享。
- **`ctx.intercept(name, config)`**：为在此上下文下方启动的插件添加服务特定的拦截配置。在返回的上下文下方加载的插件会看到 `config` 合并到服务的解析配置中（祖先条目优先）。父级上下文不受影响。

**服务解析。** Context 通过 `ctx.<serviceName>` 的方式解析服务。在 TypeScript 中，通过声明合并将服务加入 Context 接口：

```ts
declare module 'cordis' {
  interface Context { greeter: GreeterService }
}
```

声明合并为 Cordis 已声明的接口添加条目——新 `ctx.greeter` 属性的类型或事件名称。它不会生成任何运行时接线；插件必须另行提供服务或发出事件。

**低层服务存取。** `ctx.get` / `ctx.set` / `ctx.provide` / `ctx.accessor` / `ctx.mixin` 提供低层服务存储的访问和绑定能力。

**环境句柄。** `ctx.root` / `ctx.fiber` / `ctx.registry` / `ctx.reflect` / `ctx.events` / `ctx.logger` 提供对运行中上下文图的访问句柄。


## 五、Fiber（插件运行时实例）

Fiber 是一个已加载的插件实例：其生命周期状态、经过验证的配置和已注册的 effect。每个被加载的插件都拥有一个 Fiber 作用域，承载插件从声明、加载、运行到卸载的全部状态。

**Fiber 状态机。** Fiber 具有以下状态转换路径：PENDING → LOADING → ACTIVE → UNLOADING → DISPOSED，FAILED 作为异常分支从 LOADING 或 ACTIVE 进入。

| 状态 | 含义 |
|---|---|
| PENDING | 已声明，但所需依赖未就绪 |
| LOADING | 依赖已就绪，`apply` 正在运行 |
| ACTIVE | 插件正在运行 |
| FAILED | `apply` 抛出错误 |
| UNLOADING | 插件正在卸载并释放资源 |
| DISPOSED | 插件已完全卸载 |

**Fiber 的关键属性：**

- `fiber.uid`：注册表中的唯一 id；根 fiber 为 0，释放后为 `null`
- `fiber.ctx`：此 fiber 的插件运行所在的上下文（扩展父上下文）
- `fiber.config`：经过验证的插件配置（由 `update()` 更新）
- `fiber.state`：当前生命周期状态；状态转换会发出 `internal/status` 事件
- `fiber.dispose`：释放此 fiber：卸载插件，清理完成后 settle

**Effect 注册。** `ctx.effect(execute, label?)` 在此 fiber 上注册一个具有清理感知的 effect。`execute` 立即运行；它产生的 disposer 会被收集，并在返回的 disposer 被调用或 fiber 卸载时（以先到者为准）按相反顺序运行。如果 fiber 已被释放，则抛出 `CordisError('INACTIVE_EFFECT')`；如果 `execute` 返回无效形状，则抛出 `TypeError`。

**依赖驱动加载。** 如果插件的 `inject` 指定了无人提供的服务，它就会一直等待，不输出任何内容。这不是错误，因为 PENDING 是合法状态，提供方可能稍后才挂载。Cordis 在插件所需的每个服务都存在之前，将插件保持在 PENDING 状态。加载顺序无关紧要——是依赖关系而非文件顺序决定插件何时启动。

**自动清理。** 通过 `ctx` 进行的所有注册在插件卸载时都会被撤销：`ctx.on(event, handler)` 注册的事件监听器、`ctx.effect(() => cleanup)` 注册的自定义资源等。在卸载时，disposer 的调用以反向注册顺序开始，但多个异步 disposer 并发运行，没有串行完成保证。将顺序相关的清理放在从单个 `ctx.effect()` 返回的一个 disposer 中，并在其中串行 await 其步骤。

**Dispose 语义。** `fiber.dispose()` 保证：插件拥有的所有注册都被移除；子插件被递归卸载；返回的 promise 在所有异步清理完成后 resolve。


## 六、服务（Service）

Service 是 Context 服务的基类。作为插件加载的子类将自己注册为 `ctx.<name>`。Service 子类本身就是一个插件（类形式），因此 `ctx.plugin(GreeterService)` 像挂载其他插件一样挂载它。

**服务是具名能力。** 一个服务是一个插件提供、其他插件通过 `ctx` 消费的具名能力。消费者命名能力，而不是导入其提供方，因此配置可以选择提供方而不改变消费者。

**服务的类型。** 从服务提供者的角度，服务可以分为大致三种类型。第一种是由框架直接自带的服务，只要有上下文对象就可以随时访问。第二种是由框架定义但并未实现的服务，需要选择适当的插件来实现它们，在安装相应插件之前相关功能无法访问。第三种是由插件定义和实现的服务，通常需要声明这些服务作为依赖。

**Service 静态成员。** Service 类提供以下符号键：

- `Service.init`：构造后运行的实例方法的符号键（类插件）
- `Service.check`：传递给 `ctx.provide()` 的可用性谓词的符号键
- `Service.config`：phantom 拦截配置类型参数的符号键
- `Service.invoke`：使服务可调用的调用体的符号键
- `Service.extend`：派生扩展服务实例的辅助工具的符号键
- `Service.tracker`：用于上下文追踪的 tracker 元数据的符号键
- `Service.resolveConfig`：拦截配置解析辅助工具的符号键

**服务隔离。** `ctx.isolate(name, label?)` 为服务名创建独立作用域。在返回的上下文下方，`name` 服务的读写针对新标签解析，而不是父级的，因此可以提供不同的实现而不影响父作用域。两个组可以各自看到配置不同的提供方，互不影响。


## 七、事件系统（Events）

服务支持直接调用；事件让插件无需知道有哪些插件正在监听，就能发出通知。事件是 Cordis 插件之间松耦合通信的核心机制。

**声明与类型安全。** 通过 TypeScript 声明合并注册事件名称及其监听器签名：

```ts
declare module 'cordis' {
  interface Events {
    'stats/report'(name: string, count: number): void
  }
}
```

`namespace/action` 命名约定让扁平的事件命名空间保持易读。

**五种分发模式。** 每个事件具有以下分发模式之一，且只能通过对应方法分发。分发模式是事件公开约定的一部分：

| 模式 | 调用 | 语义 |
|---|---|---|
| emit | `ctx.emit(name, ...args)` | 同步广播；不会等待或收集返回的 promise 与值 |
| parallel | `await ctx.parallel(name, ...args)` | 所有监听器并发运行，并一同等待 |
| serial | `await ctx.serial(name, ...args)` | 监听器按顺序运行并等待；第一个非 null/false/undefined 返回值胜出 |
| bail | `ctx.bail(name, ...args)` | serial 的同步版本 |
| waterfall | `ctx.waterfall(name, ...args, next)` | 环绕中间件 |

**Waterfall 语义。** `ctx.waterfall` 是环绕中间件。监听器接收 `(...args, next)`。调用 `next()` 会执行下游监听器；下游返回值通过 `next()` 返回当前包装层，可由该层包装后继续向外返回。不调用 `next()` 直接返回则短路——这是设计意图，用于实现拦截和网关行为。协作式监听器通常修改一个共享的请求或决策对象，然后委托。监听器也可以选择完全替换结果，下游监听器将只看到替换后的结果。仅当监听器必须在普通注册之前运行时才使用 `prepend: true`。对于单决策事件，短路是设计意图。策略监听器在拥有决策权时可以不调用 `next()` 直接返回，而仅做标注或观察的监听器则必须委托。

**事件监听器的可逆性。** `ctx.on()` 属于 effect，监听器会随插件一同消失，绝不需要手动维护 `removeListener`。


## 八、生命周期与可逆副作用

Cordis 插件可以被配置编辑、热重载、显式释放或所需服务丢失等方式卸载。通过 Cordis API 进行的注册是 effect，当拥有它的插件卸载时会被撤销。

**Effect 的回滚语义。** `ctx.effect(execute)` 在 fiber 上注册具有清理感知的 effect。`execute` 立即运行；它产生的 disposer 会被收集，并在返回的 disposer 被调用或 fiber 卸载时（以先到者为准）按相反顺序运行。调用 disposer 两次是 no-op。

**每个注册都应有对应的 disposer。** 要么从 `ctx.effect()` 返回一个，要么使用 Cordis 提供的辅助方法自动处理。如果 teardown 顺序有要求，请将相关工作放在同一个 effect 中，以确保资源按预期顺序释放。

**依赖回滚链。** 提供方被卸载时，所有依赖它的插件自动卸载（effect 回滚）；新提供方就绪后，自动重载。依赖方不需要写任何重连代码。

**实践规则。** 将行为封装为插件。拦截和策略优先使用事件；直接能力调用优先使用服务方法。


## 九、注册表（Registry）

Registry 负责插件加载和依赖注入。

**`ctx.plugin(plugin, ...args)`** ：在当前上下文中加载插件。接受函数、类或 `{ apply }` 对象插件。`args` 是根据其 `Config` schema 验证的插件配置。返回 fiber；await 它在加载完成后 settle（在配置或启动错误时 reject）。

**`ctx.inject(deps, callback)`** ：在请求的服务可用后运行回调。是 `ctx.plugin({ inject, apply: callback })` 的简写：每当所需服务变化时，回调被卸载并重新运行。返回 fiber；await 它在加载完成后 settle。

**插件的运行时记录。** 每个插件回调在所有 fiber 之间共享一个可变的注册表记录，包含：`name`（从第一个注册的插件形态复制）、`fibers`（此插件的每个活跃 fiber，每次 `ctx.plugin()` 调用一个）、`callback`（所有 fiber 共享的可执行入口点）。


## 十、设计哲学

Cordis 的内核只做三件事，把复杂度锁死在插件层、把稳定性锁死在内核层：

**生命周期管理。** 加载一行 `ctx.plugin(Plugin, options)`，背后发生了三件事：Cordis 为这个插件创建一个子 Context（也就是一个作用域）；执行插件的 setup 函数，把 `ctx` 传进去；插件在 setup 里通过 `ctx` 注册事件、服务、命令——这些注册项全部属于这个子 Context。

**插件解析。** Cordis 负责处理插件之间的依赖关系，确保插件在所需服务就绪后才加载，在服务消失时自动卸载。

**上下文传播。** Cordis 负责管理插件间的数据共享，通过 Context 代理实现作用域化的服务解析和事件传播。

Cordis 是一个容器，而不是一个框架。框架会递给你勺子：“来，用我的 Router 写路由，用我的 Command 写命令。”容器不递勺子，它只负责把插件装进去、把依赖理清楚、把上下文传下去。至于勺子长什么样，是插件自己的事。内核不碰业务逻辑，它只是让插件们能“装得上、理得清、传得开”。


## 十一、架构总结

Cordis 的核心架构可以概括为以下层次关系：

- **Context** 是核心对象，作为服务的容器和访问入口。通过 `extend()`、`isolate()`、`intercept()` 创建作用域化子上下文，通过声明合并实现类型安全的服务与事件访问。
- **Fiber** 是插件的运行时实例，跟踪生命周期状态、验证配置和已注册的 effect。Effect 在 fiber 卸载时自动回滚，保证可逆性。
- **Service** 是具名能力，通过 Context 暴露，通过 `inject` 声明依赖。支持 `isolate()` 实现服务隔离。
- **Events** 提供五种分发模式（emit、parallel、serial、bail、waterfall），支持类型化声明和自动清理。
- **Registry** 管理插件加载和依赖注入，`ctx.plugin()` 和 `ctx.inject()` 是核心入口。

整个框架的设计哲学是**可逆的、动态的、声明式的**：插件声明其依赖和副作用，Cordis 负责在正确的时机挂载和卸载它们。注册即副作用，卸载即回滚。这使得系统可以在运行时安全地重组，而不需要手动管理复杂的启动和清理序列。

Cordis 的架构与 Koishi 一脉相承。Koishi 的大部分特性都是围绕上下文进行设计的——即使不同的上下文可以隶属于不同的插件、配置不同的过滤器，但许多功能在不同的上下文中访问的效果是一致的。应用可以被理解成一个容器，搭载了各种各样的功能，而上下文则单纯提供了一个接口来访问它们。这种组织形式被称为服务。对于已经有 IoC / DI 概念的同学来说，服务就是一种类似于 IoC 的实现。Koishi 正是构建在 Cordis 之上的集大成者，Cordis 提供骨架（生命周期与依赖），其余能力以插件形式组合。