# ct-test：host 全链路测试插件

状态：⚪ 已归档（测试完成，插件已删除；文档仅作链路验收记录）

> 目标：用**一个测试插件**打通「UI 槽位替换 → 前后端 API → persist 落盘 → 刷新回读」全链路。
> 范围：**单一备忘录、多条目**（可增改删条目，不新建备忘录）。
> UI 结构与样式**打包进插件包**；不修改 `ct-ui-layout`。

## 1. 包身份

| 项 | 值 |
|---|---|
| 包名 | `@cordis-tavern/ct-test` |
| 插入 id | `test` |
| `ct.bundle.path` | `./dist/cordis.patch.yml` |
| Node `inject` | `['webServer', 'persist']` |
| 前端 | `exports["./client"]`；`ct.client`: `platform: web`，`immediately: true`，`inject: ['@cordis-tavern/ct-ui-slots']` |
| factory | `{ name, apply, inject: ['slots'] }` |

## 2. UI：结构与样式进包

- 来源：`deepseek_html_20261008_6af55c.html` 的**结构 + 样式**一并搬入插件包（如 `assets/memo/`），构建进 client 产物。
- **禁止**引用 `D:\Users\...` 等本机路径。
- 槽位尺寸以布局为准：**topbar 高 40px**（HTML 内若有 60px 一律改为 40）；中部门 45% / ≤1000px 满页等舞台样式与布局一致。

### 2.1 槽位策略（不占 root、不改 layout）

| 槽位 | 策略 | 本测试 |
|---|---|---|
| `main` | `single`，**以权重替换**：`register({ name: 'main', id: 'ct-test', kind: { single: true, priority: 1 } }, Memo)`（盖过 layout 在 `main` 上的占位） | 备忘录主区挂这里 |
| `topbar` | `list` | **目前无作用**，本测试不注册 |
| `root` | 由 `ct-ui-layout` 占据 | 不动 |

### 2.2 备忘录行为（单一备忘录 · 多条目）

- 「＋ 新建」= **条目**新建，不是新备忘录。
- 条目：文本，可编辑、可删除；**无**多备忘录切换 / 新建备忘录。
- 最小交互：增条目 / 改文本 / 删条目 / **保存** / 启动或刷新时**加载**。

## 3. 前后端 API（`webServer.register`，exact）

| 方法 | 路径 | 请求 | 响应 |
|---|---|---|---|
| `GET` | `/api/memo/load` | — | `200` `{ items: [{ id: string, text: string }] }`；无数据 `items: []` |
| `POST` | `/api/memo/save` | `{ items: [{ id: string, text: string }] }` | `200` `{ ok: true }`；非法 body → `400` `{ error }`；persist 失败 → `500` `{ error }` |

- 条目顺序 = `items` 数组序。
- handler 自管 `res.end`；错误必须带响应体（禁止静默失败）。

## 4. 持久化（ct-persist）

`namespace: 'beiWangLu'`（落在 `$CT_HOME/data/beiWangLu/`）：

| key | 文件 | 内容 |
|---|---|---|
| `index` | `data/beiWangLu/index.json` | `{ ids: string[] }` 条目顺序 |
| `item-<id>` | `data/beiWangLu/item-<id>.json` | `{ id, text }` 单条内容 |

保存：写各 `item-<id>` → 写 `index`。加载：读 `index` → 按序读 item；缺项跳过。

## 5. 验收

1. `ct web` 后主区为备忘录（`main` 权重替换生效）。
2. 增/改/删条目 → 保存 → `POST /api/memo/save` 返回 `200`。
3. `data/beiWangLu/index.json` 与 `item-*.json` 存在且 JSON 合法。
4. 刷新后 `GET /api/memo/load`，条目与顺序与保存前一致。
5. 非法 body → `400` + `{ error }`；persist 失败 → `500` + `{ error }`。
6. 无本机 Downloads 依赖；干净 `$CT_HOME` 可跑通。

## 6. 依赖

| 依赖 | 用途 |
|---|---|
| `ct-ui-slots` | `register` 到 `main` |
| `ct-host-web` | `webServer` |
| `ct-persist` | `ctx.persist.json` |
| `ct-ui-layout` | 提供舞台与 `main` 槽位（本测试不改它） |
| `ct-client-cordis` | 浏览器装载 factory |

## 7. 明确不做

- 修改 `ct-ui-layout` / 槽位结构 / `renderSlot` 接线
- 使用 `topbar`（list）
- 多备忘录、主题、HMR、鉴权、远程同步
