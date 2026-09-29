# persist 插件设计文档（v1）

状态：🟢 已实现（`@cordis-tavern/ct-persist@0.0.1-alpha.2`；`data/` 固定前缀只追加）

## 1. 目标

为插件提供统一的数据管理服务。  
v1 版本仅支持 JSON 文件持久化。

- 服务名：`persist`
- 操作函数：`json(path, options?)`
- 调用形态：`ctx.persist.json(...)`
- 默认根目录：`$CT_HOME`

---

## 2. API 签名

```ts
ctx.persist.json<T>(path: PathOptions, options?: JsonOptions<T>): Promise<...>
```

### PathOptions

```ts
interface PathOptions {
  key?: string
  namespace?: string
}
```

- `path.key`：逻辑键名，对应真实文件名，如 `setting` 对应 `setting.json`
- `path.namespace`：命名空间，**追加在固定 `data/` 之下**（`data` 不被覆盖、只被追加）。缺省或 `''` → `data/`；`'user'` → `data/user/`

> `list` 操作不需要 `path.key`，只使用 `path.namespace`。

### JsonOptions

```ts
interface JsonOptions<T> {
  op?: 'read' | 'write' | 'update' | 'remove' | 'has' | 'list'
  default?: T
  value?: T
  updater?: (current: T | undefined) => T | undefined
}
```

- `options.op`：具体操作，默认值 `read`
- `options.default`：`read` 专用，文件不存在时的默认返回
- `options.value`：`write` 专用，写入值
- `options.updater`：`update` 专用，先读后写

---

## 3. 操作与返回值

| 操作   | `op`      | 专用参数       | 返回                        | 说明                                  |
| ---- | --------- | ---------- | ------------------------- | ----------------------------------- |
| 读    | `read`，默认 | `default?` | `Promise<T \| undefined>` | 文件不存在时返回 `default`，否则返回 `undefined` |
| 写    | `write`   | `value`    | `Promise<void>`           | 覆盖写入 JSON                           |
| 更新   | `update`  | `updater`  | `Promise<T \| undefined>` | 读当前值，传给 `updater`，再写回               |
| 删除   | `remove`  | 无          | `Promise<boolean>`        | 成功删除返回 `true`，不存在返回 `false`         |
| 判断存在 | `has`     | 无          | `Promise<boolean>`        | 文件存在且为文件时返回 `true`                  |
| 列出键  | `list`    | 无          | `Promise<string[]>`       | 列出指定命名空间下所有 JSON 文件的逻辑键             |


---

## 4. 调用示例

```ts
// 读：$CT_HOME/data/config.json
await ctx.persist.json({ key: 'config' })

// 读，带默认值
await ctx.persist.json(
  { key: 'config' },
  { default: { enabled: true } }
)

// 写
await ctx.persist.json(
  { key: 'config' },
  { op: 'write', value: { enabled: false } }
)

// 更新
await ctx.persist.json(
  { key: 'counter' },
  { op: 'update', updater: n => (n ?? 0) + 1 }
)

// 删除
await ctx.persist.json(
  { key: 'config' },
  { op: 'remove' }
)

// 判断存在
await ctx.persist.json(
  { key: 'config' },
  { op: 'has' }
)

// 列出 data 命名空间下所有 key
await ctx.persist.json(
  { namespace: 'data' },
  { op: 'list' }
)

// 使用自定义命名空间
await ctx.persist.json(
  { key: 'user', namespace: 'data/user' }
)
// 实际读取 $CT_HOME/data/user/user.json
```

---

## 5. 路径规则与安全性

真实路径计算：

```txt
真实路径 = $CT_HOME / data / path.namespace / path.key + ".json"
```

**`data/` 是固定前缀，不被覆盖、只被追加**：

| `path.namespace` | 真实目录 |
| --- | --- |
| 缺省 / `''` | `$CT_HOME/data/` |
| `'user'` | `$CT_HOME/data/user/` |
| `'data/user'` | `$CT_HOME/data/data/user/`（原样追加，不识别「已是 data」） |

因此：

```ts
await ctx.persist.json({ key: 'user' })
// 实际读取 $CT_HOME/data/user.json

await ctx.persist.json({ key: 'user', namespace: '' })
// 实际读取 $CT_HOME/data/user.json

await ctx.persist.json({ key: 'user', namespace: 'user' })
// 实际读取 $CT_HOME/data/user/user.json
```

安全要求：

1. `path.namespace` 与 `path.key` 必须做路径穿越校验。
2. 拒绝 `..`、绝对路径、Windows 盘符、空字节、路径分隔符进入 `path.key`。
3. 最终路径必须通过 `path.resolve` 与 `path.relative` 确认仍位于 `$CT_HOME` 内。
4. 必要时对父目录做 `realpath` 检查，防止符号链接逃逸。
5. `path.namespace` 统一视为相对 `$CT_HOME/data` 的子路径（可为空），不建议以 `/` 开头。

---

## 6. 并发与原子性

`update` 是“先读后写”，多插件并发时会出现丢失更新。建议：

- 按文件路径维护 Promise 队列或锁。
- `write` 与 `update` 使用原子写：写临时文件，再 `rename` 覆盖。
- `update` 在锁内完成读、改、写。
- 返回值语义：为`undefined`时说明覆盖写入未找到目标

---

## 7. 错误语义

| 场景                | 行为                              |
| ----------------- | ------------------------------- |
| 文件不存在，`read` 无默认值 | 返回 `undefined`                  |
| 文件不存在，`read` 有默认值 | 返回 `default`                    |
| JSON 解析失败         | 抛出错误，不要静默返回默认值                  |
| 权限不足、磁盘错误         | 抛出错误                            |
| `remove` 文件不存在    | 返回 `false`                      |
| `has` 目标是目录       | 返回 `false`                      |
| `list`            | 只列 `.json` 文件，返回不带扩展名的 key，建议排序 |

---