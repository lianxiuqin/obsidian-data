# ct-ui-renderer

状态：🟢 已实现（`0.1.0-rc.1`）

> 渲染器插件：provide **`uiRenderer`**，由 **`onAllReady()`** 主动把根插槽挂成 React 树。

## 职责

| 项 | 说明 |
| --- | --- |
| 服务 | `provide: ['uiRenderer']`，暴露 `onAllReady()` |
| 渲染时机 | **被调用时**才渲染；apply 只挂服务，不自动渲染 |
| 渲染 | `createRoot(container).render(<Root />)`，`Root` 内 `slots.renderSlot('root', owner)` |
| 挂载容器 | 默认 `#root`（config `container`）；找不到容器抛错指名 selector |
| React | `require('react')` / `require('react-dom/client')`（`__CT_MODULES__`，onAllReady 时解析） |
| slots 访问 | `inject: ['slots']`（属性读取合法）+ `reflect.get('slots')` 兜底 |

## 配置

| 键 | 默认 | 说明 |
| --- | --- | --- |
| `container` | `#root` | CSS selector |
| `owner` | `{}` | 传给 `renderSlot('root', owner)` |

## 总结

ct-ui-renderer 在 `onAllReady()` 时把登记内容渲染为 React 树挂到页面容器；激活不依赖全局插件齐备，也不在 inject 就绪时自动渲染。
