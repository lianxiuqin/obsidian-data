# ct-ui-layout

状态：🟢 已实现（`0.1.0-rc.1`，V1 机制验证）

## 目标

最简布局页面：验证 root 插槽接管与 React 挂载整链。

## 行为

```ts
inject: ['slots']
slots.register({ single: true, priority: 1 }, 'root', Layout)
// Layout 按大小居中显示 cordis-tavern-test
```

| 项 | 值 |
| --- | --- |
| 插槽 | `root` / `single` / `priority: 1` |
| 文案 | `cordis-tavern-test` |
| 字号 | 有 `width/height` 时按短边估算（16–96px）；否则 `clamp(16px, 8vmin, 96px)` |

## 包形态

- 双入口；Node 空 `apply`；`insert id: uiLayout`
- factory：`{ name, apply, inject: ['slots'] }`

## 总结

ct-ui-layout 接管 `root` 显示测试文案，是首个走通 client-cordis → slots → renderer 整链的 UI 插件。
