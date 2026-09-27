---
tags:
  - 插件开发
  - host
  - CLI
  - 启动入口
---

# ct-host-web v3

状态：🟡 设计已确认（待实施）

> 承接 [[ct-host-web-v2|v2]]。本页为 v3 设计：把 **`ct <profile>`** 做成启动入口。
> 实际落点在**内核** `cordis-tavern` 的 CLI / 启动层，**不在** host 插件包本身；
> 本文档按用户裁定只作 v3 回写，**不同步** [[项目文档/项目设计文档/内核设计/CLI层|CLI层]] / [[项目文档/项目设计文档/内核设计/启动层|启动层]] 正典。

## 目的

目标创建命令，`ct <profile>` 作为启动入口。示例：`ct web` —— 在 `web` profile 内启动 bootstrap，加载该环境的内部插件。

## 已裁定（用户）

| 项 | 裁定 |
| --- | --- |
| 调度优先级 | **未命中注册表命令 → 把 `argv[0]` 当 profile 名启动**。命中则仍走命令（`init` / `plugin` / `update` 及插件命令优先）。 |
| 启动后行为 | `start()` 就绪后**保持运行直到 Ctrl+C**。 |
| 文档范围 | **只回写本文件**；不同步 CLI层 / 启动层设计文档。 |

## 行为

```
argv[0]
  ├─ 注册表命中（环境级 → bootstrap 级）→ 既有命令分发（不变）
  └─ 未命中 → 当作 profile 启动：
       1. 未初始化（无 CT_HOME）→ 报错并提示 ct init（与 ct plugin 同文案）→ exit 1
       2. 环境名形状非法 → 经 profileDir 守卫报错 → exit 1
       3. profile/<name>/ 不存在 → 报错指名路径 → exit 1
       4. 额外参数（如 ct web extra）→ 报错 → exit 1
       5. start({ env: name }) 装配并装载该 profile 插件
       6. 打印就绪信息 → 挂起（Ctrl+C 退出）
```

- 拼错的命令名不再得到 `unfound command:`，而是「环境不存在 / 未初始化」——用明确文案区分，避免与「命令未注册」混淆时完全无声。
- 空 `ct.profile` 且目录存在：合法，启动空系统后仍挂起（与启动层「空表是合法输入」一致）。

## 分层

| 方向 | 是否允许 |
| --- | --- |
| CLI → 启动层 `start()` | ✅ 本轮新增 |
| 启动层 → CLI | ❌ 仍禁止 |

启动层 `StartOptions.env` 注释中的升级触发条件本轮兑现：**env 来自 argv 时，路径守卫加在 CLI 入口并复用 `profileDir()`**，启动层自身仍不校验显式 `env`。

## 落点

| 动作 | 文件 |
| --- | --- |
| 改 | `bootstrap/cli/index.ts` —— `findCommand` 未命中时走 profile 启动，不再直接 `unfound command` |
| 新增 | `bootstrap/cli/commands/start.ts`（或等价内部模块）—— 守卫 + `start()` + 就绪挂起；**不**写入 `commands.yml` |
| 改 | `test/cli.test.mjs` —— 分发与守卫用例 |

发布构建：启动逻辑随 `dist/cli/index.js` 打进 bundle（esbuild `bundle: true`），**不必**成为 `build-dist.mjs` 的独立 ENTRIES，也不进注册表自检。

## 错误语义

| 情况 | 行为 |
| --- | --- |
| 未初始化 | `项目未初始化（.env 中没有 CT_HOME），请先执行 ct init。` exit 1 |
| 非法环境名 | `profileDir` 既有文案，exit 1 |
| 环境目录不存在 | 指名 `profile/<name>` 路径，exit 1 |
| 多余参数 | 报错并说明 `ct <profile>` 不接受额外参数，exit 1 |
| `start()` 抛错 | 消息打到 stderr，exit 1（沿用调度器 catch） |

## 测试要点

1. 未初始化 → 提示 `ct init` + exit 1（不进入 start）
2. 非法环境名（`..`、含分隔符等）→ 拒绝
3. 环境不存在 → 指名路径 + exit 1
4. 注册表命中仍优先于 profile（`init` / `plugin` 不被当成环境）
5. 额外参数 → 报错
6. （可注入 `start` 时）守卫通过后调用 `start({ env })` 的接缝

## 明确不做

- 不改启动层 `start` / `assembleTable` 契约
- 不改 `commands.yml`、不注册名为 `web` 的假命令
- 不做热重载、不做出参 flags（`--open` 等）
- 不同步 CLI层 / 启动层正典文档

## 总结

v3 把 CLI 的「未命中命令」路径升级为 **profile 启动入口**：`ct web` 经 `profileDir` 守卫后调用启动层 `start({ env: 'web' })`，装载该环境内部插件并保持进程存活；注册表命令优先级不变，守卫落在 CLI 侧以兑现启动层关于 argv 派生 env 的升级条件。
