# CLI层

状态：🟢 已实现（`bootstrap/cli/` 与 `test/cli.test.mjs` 可核对）

## 概述

CLI 层是 cordis-tavern 的命令行入口，发布为 `ct` 命令（`package.json` 的 `bin: { ct: "dist/cli/index.js" }`）。

它做三件事：

1. **调度命令**：把 `argv` 分派到一个命令实现文件，由该文件自己处理后续参数
2. **管理环境**：`ct init` 建立 `$CT_HOME` 骨架
3. **管理插件**：`ct plugin add / remove / update / list` —— 通过 pnpm 安装插件、登记 `ct.profile`、并重建命令表

源码入口 `bootstrap/cli/index.ts`。设计模型是 **git 式调度**：主 CLI 只持有「命令名 → 处理函数（文件）」的表，**子命令由命令自己的文件分发**，主文件不管子命令语义。

> **与启动层的分工**：CLI 负责**装** —— `ct init` / `ct update` 把内核装进 `$CT_HOME`、`ct plugin` 维护 `ct.profile`；启动层负责**用**。两者各自持有一份 `ct.profile` 与两层 patch 的**镜像解析实现**，**两处必须同改**。

> **约束**：本层源码由 Node 原生类型剥离直接执行，只能使用**可擦除语法**（禁 `enum` / `namespace` / 构造函数参数属性），相对导入必须带 `.ts` 扩展名。

---

## 调度器

### 数据流

```
argv → 命令名 + 参数
     → 两级注册表查找（环境级遮蔽 bootstrap 级）
     → 解析 file（相对注册表所在目录）
     → 动态 import 该文件
     → 调用其 default(args)
```

### 定位权威是入口，不是本层的相对上溯

`BASE_DIR` 由**入口**（`bootstrap/cli/index.ts`）用 `import.meta.url` 的目录显式算出，再传给注册表查找。

原因：`bootstrap/cli/`（源码布局）与 `dist/cli/`（发布布局）里入口都与 `commands.yml` 同级，这是唯一在两种布局下都成立的定位方式。注册表模块里那条「相对本文件上溯一级」的推导**只在源码布局成立**——打成单文件 bundle 后会落到 `dist/`，找不到 `commands.yml`，CLI 静默退化成一个没有任何内置命令的空壳（表现为 `（当前没有可用命令）`，`ct init` 报 `unfound command: init`）。该推导仅作为源码布局下的默认值保留。

### 错误语义与退出码

| 情况 | 行为 |
|---|---|
| 无参数 | 打印用法与可用命令，exit 1 |
| 未命中命令 | `unfound command: <name>`，exit 1 |
| 注册项缺 `file` 字段 | 抛错，exit 1 |
| 命令文件不存在 / 不是默认导出函数 | 抛错，exit 1 |
| 命令实现抛错 | 消息打到 stderr，exit 1 |
| 正常完成 | exit 0 |

用法输出由注册表条目生成（`explain` 存在时附在命令名后）。

---

## 命令注册表

### 文件格式

`commands.yml` 是一个数组，每项：

| 字段 | 必填 | 含义 |
|---|---|---|
| `command` | ✅ | 命令名，调度时按它精确匹配 |
| `file` | ✅ | 命令实现文件，**相对注册表所在目录**解析 |
| `explain` | ❌ | 用法列表里的说明 |
| `id` | ❌ | 该命令归属的服务 id（来自插件的 `ct.bundle`）。判定在重放时于内存中完成；写进文件是为了人工阅读与错误消息，**条目在文件里的先后不承载「同 id」语义** |
| `subcommands` | ❌ | 子命令组，**原样带出，本轮不分发** |

### bootstrap 级（兜底）

`bootstrap/cli/commands.yml`：

```yaml
- command: init
  explain: 对项目进行初始化
  file: commands/init.ts
- command: plugin
  explain: 管理插件（add / remove / update / list）
  file: commands/plugin.ts
- command: update
  explain: 升级数据文件夹里的内核副本
  file: commands/update.ts
```

### 环境级（优先，遮蔽 bootstrap 级）

`$CT_HOME/profile/<当前环境>/commands.yml`。由 `ct plugin add / remove / update` 通过**全量重放**生成，条目的 `file` 是插件命令模块的**绝对路径**。

- 查找顺序：环境级 → bootstrap 级，**命中即止**（实现遮蔽语义）。
- `CT_HOME` 未设置时**跳过环境级** —— 未初始化状态下调度器必须可用，否则 `ct init` 鸡生蛋。

> **当前环境从哪来**：读仓库根 `.env` 的 **`CT_PROFILE`**，缺省 `web`（见[[项目文档/项目设计文档/内核设计/存储层|存储层]]）。因此 `ct plugin add -p dev` 装出来的插件，其命令写进 `profile/dev/commands.yml` 并被 CLI 读到 —— 安装与查找用的是同一个环境名。
>
> `CT_PROFILE` **不由 `ct init` 写入**；形状非法（`.` / `..` / 含路径分隔符或盘符）**直接报错**，而不是静默退回 `web`。
>
> 这个字段的直接来由：此前环境级路径**硬编码 `web`**，于是装进其它环境的插件「装得上、却调不到」，而且不报错。

### 写回

`writeRegistry` 用统一的 YAML 选项写文件（`lineWidth: -1` 不折行）：默认 80 列会把长绝对路径折断成多行标量，读回来仍正确，但人读不了、diff 变噪声。空表写成 `[]`（与 `ct init` 建的命令表格式一致）。

---

## 内置命令

### `ct init`

对项目进行初始化。**不接受任何参数**。

流程：

1. **幂等短路**：`.env` 里已有非空 `CT_HOME` → 打印「项目已经初始化」后 exit 0
2. **交互询问**数据保存地址，提示含空格路径用双引号包裹
3. **校验**（不合法则重新询问）：非空、无 Windows 保留字符（盘符后的冒号不算非法）、拒绝盘符相对写法（`C:foo`）、**地址不得落在任何 pnpm workspace 之内**（见下）
4. **目标校验**：目标必须是**不存在**或**已存在的空目录**
5. `pnpm init`（在目标目录内）
6. **删掉 `devEngines` 整块**：`pnpm init` 写入的是范围值（`^11.22.0`），而 pnpm 11+ 要求精确 semver，会让该目录内任何 `pnpm install` 必然失败
7. **建骨架**：

```
$CT_HOME/
├── cordis.patch.yml                  # []（用户级配置覆盖）
└── profile/web/
    ├── package.json                  # {"ct":{"profile":[]}}
    ├── cordis.patch.yml              # []（环境级配置覆盖）
    └── commands.yml                  # []（环境级命令注册表）
```

8. **装配内核**：把 `bootstrap/cli/kernel.json` 里的 5 个内核包按**精确版本**合并进 `$CT_HOME/package.json` 的 `dependencies`，然后在该目录跑 `pnpm install`；装完**复核**每个包自己的 `package.json` 版本是否等于目标版本（判据是包自身，不是 `$CT_HOME/package.json` 写了什么 —— 两者分叉正是「写了却没装上」）。
9. **行级改写**仓库根 `.env` 的 `CT_HOME`（存在则替换该行，否则追加；其余行原样保留）

**为什么 `ct init` 要装内核**：内核（cordis / plugin-loader / plugin-include / schemastery / cosmokit）**只存一份**，就住在数据文件夹里；宿主与插件都从它解析，单实例因此由构造保证（见[[项目文档/项目设计文档/内核设计/启动层|启动层]]的「内核根」）。主包**不**把内核声明为运行时依赖 —— 否则下游装主包时会拉进第二份。

> ⚠ 步骤 8 需要联网（内核从 registry 下载一次）。装不上会**抛错且不写 `.env`**：宁可没初始化，也不要留下一个指向半成品数据文件夹的项目。

> ⚠ **为什么地址里要查 pnpm workspace**：步骤 8 要在 `$CT_HOME` 里跑 `pnpm install`，而 `runPnpm` 的 workspace 守卫会在 spawn 前拒绝落在 workspace 内的目录。若把这道校验留到那一刻，失败发生在**目录已建好、`pnpm init` 也跑过之后** —— 用户拿到的是「请删除该目录再重试」。所以它被**提前到询问阶段**（判据与守卫同一个 `findWorkspaceRoot`）。
>
> [实测] 这不是假想：pnpm 12 的 `pnpm add` 会在**项目根**写下 `pnpm-workspace.yaml`（放 `minimumReleaseAgeExclude` 之类的供应链策略），于是**任何用 pnpm 装过本包的项目，其 `./data` 都会落在 workspace 之内**。数据文件夹要选在项目之外（推荐同级的兄弟目录）。

> ⚠ `$CT_HOME` 的值必须落在**任何 pnpm workspace 之外**。本仓库根自带 `pnpm-workspace.yaml`，故 `./ct` 这类指向仓库内部的值会被 workspace 守卫拒绝。推荐与仓库同级的兄弟目录，见[[项目文档/项目设计文档/内核设计/存储层|存储层]]。

### `ct update`

升级**数据文件夹里的内核副本**。**不接受任何参数**。

```
  （无输出 = 无差异）  内核已是最新（5 个包）: <$CT_HOME>
  @cordis-tavern/cordis: 4.0.0-rc.10 → 4.0.0-rc.11
  ...
  已更新内核: <$CT_HOME>
```

流程：读 `.env` 的 `CT_HOME`（未初始化 → 报错）→ 守卫目录存在（不存在时给的是「CT_HOME 指错了」而不是裸 ENOENT）→ 读 `kernel.json` → 逐个读 `$CT_HOME/node_modules/<包>/package.json` 得到**已装版本** → 与目标比对 → 无差异即结束（**不触网**）→ 有差异则合并写回 `$CT_HOME/package.json` → `pnpm install` → **复核**是否真的就位。

它**不动** `ct.profile`、环境目录与插件包 —— 插件的升级是 `ct plugin update`。


### `ct plugin`

```
用法: ct plugin <子命令> [选项] [参数]

  ct plugin add    -g | -p <环境名> <插件名>     安装并登记插件
  ct plugin remove -g | -p <环境名> <插件名>     卸载并撤销登记
  ct plugin update -g | -p <环境名> <插件名>     升级已登记的插件
  ct plugin list   <环境名>                      列出该环境的 ct.profile

  -g  全局范围（$CT_HOME）    -p  指定环境（$CT_HOME/profile/<环境名>）
```

**作用域与安装根**（由主命令解析后交给子命令，子命令不再自己拼路径）：

| 选项 | 安装根 = pnpm 的 cwd |
|---|---|
| `-g` | `$CT_HOME` |
| `-p <环境名>` | `$CT_HOME/profile/<环境名>` |

`-g` 与 `-p` 必须**恰好指定一个**。

**`add` 与 `update` 的差别只有两条守卫**，且都由子命令名就地派生：

| 子命令 | 环境不存在时 | 要求已登记 |
|---|---|---|
| `add` | 自动建骨架 | 否 |
| `update` | 报错 | 是 |

**动词判据**：`installedHere = declaredIn(installRoot, pkg)` —— pnpm 收到的是 `update` 还是 `install`，看的是「它**装在这个安装根**里吗」，**不是**「它登记在列吗」。两者是两件事，混用会静默做错动作。

**执行顺序（不变式）**：

```
全部校验前置 → 唯一外部副作用（pnpm install/update/remove）
             → 在内存里算完所有环境的计划（登记表 + 命令表）
             → 统一落盘
```

先算完再落盘保证原子性：否则第二个环境失败会留下「第一个环境的 `ct.profile` 已改、命令表没写」的半成品。

**`list`**：逐行打印该环境的 `ct.profile`；环境不存在则报错。路径**必须**经 `profileDir()` 构造——它是唯一带穿越守卫的路径构造者，自行 `path.join` 会让 `ct plugin list ../..` 读到 `$CT_HOME` 的父目录且不报错。

**`remove`**：两种「找不到」分开报 —— 「不在 `ct.profile` 里」（没登记）与「不在安装根的 `package.json` 依赖表里」（没装在这个根）。不分会让用户拿到 pnpm 的 `ERR_PNPM_CANNOT_REMOVE_MISSING_DEPS`，不可行动。

---

## 插件自定义命令

插件通过 `package.json` 的 `ct.commands` 声明命令（见[[项目文档/项目设计文档/插件设计/插件契约层|插件契约层]]）：

```json
{
  "ct": {
    "commands": [
      { "command": "build", "path": "./commands/build.js", "explain": "构建当前插件" }
    ]
  }
}
```

### 入口文件只接受 `.js` / `.mjs`

| 扩展名 | 结果 | 原因 |
|---|---|---|
| `.js` / `.mjs` | ✅ | 调度器动态 import 的目标 |
| `.ts` | ❌ | `node_modules` 下 Node 拒绝剥离类型（`ERR_UNSUPPORTED_NODE_MODULES_TYPE_STRIPPING`） |
| `.cjs` | ❌ | default 契约与调度器的「必须是函数」不一致 |
| `.JS` / `.MJS` | ❌ | 比对大小写敏感，Node 会在调度点抛 `ERR_UNKNOWN_FILE_EXTENSION` |

**拒绝必须发生在登记点**：放行等于把失败从 `ct plugin add` 推回 `ct` 的调度点，届时错误与「命令根本没注册」无法区分。

### 命令表全量重放

命令表不是增量维护的，而是**每次都全量重算**：

```
命令表 = f(ct.profile 顺序, 环境目录, $CT_HOME)
```

`rebuildTable` 对登记表里每个名字：定位包目录 → 读契约 → 取 `insert[].id` 的第一个 → `table.filter(e => e.id !== id)` → 追加该插件的命令。两行合并即实现全部四类规则：

- **同 id 后来者居上**：过滤掉前者的条目
- **同 id 后来者无 `ct.commands`**：该 id 一条都不剩
- **前辈浮现**：移除后来者后，`ct.profile` 里更早的同 id 插件重新被重放出来
- **跨 id 同名命令 → 报错**：不同 id 的两个插件声明同名 `command` 时抛错并指名两个 id。调度器取首个匹配，静默遮蔽不可接受
- **同一插件内重复声明同名命令** → 报「重复声明」而非「两个插件」

### 子命令不分发

`subcommands` 只声明「名字 + 说明」、**没有 `path`**：主命令文件自己分派子命令（git 式调度模型）。注册表把它们原样带出，CLI 不做任何子命令层的解析。

---

## 信任边界（5 处）

| 边界 | 守卫 | 位置 |
|---|---|---|
| **插件名** | 必须是合法 npm 包名（正则：可选 scope，其余只允许小写字母 / 数字 / `-` / `_` / `.` / `~`）；只接受**裸包名**，不接受路径、版本号或其它 spec 形态 | `parsePluginArgs` |
| **环境名** | 进路径 → 交给 `profileDir` 的穿越守卫（判据是**结果**：目标必须是 `$CT_HOME/profile` 的直接子目录；外加「环境名不得自带路径根」的结构性判据，挡 `D:evil` 这类形式） | `profileDir` |
| **选项形态** | 环境名与插件名都不得以 `-` 开头 —— 否则 `ct plugin add -p -g x` 会把 `-g` 当成环境名，**静默装错作用域** | `requireNotOption` |
| **注册表 / 契约文件** | 格式非法即抛错并指名文件与位置（缺 `command` / `file`、非数组、非对象…）；**契约是信任边界，不静默误解** | `loadRegistry` / `readContract` |
| **pnpm 调用** | 插件名会原样进 pnpm 的 `argv`（Windows 上经 `cmd.exe /c` 调用），故入口必须拒绝一切会被 `cmd.exe` 重解析的载荷 | `PACKAGE_NAME` + `runPnpm` |

### pnpm 调用的两个平台坑

1. Windows 上 pnpm 是 `.cmd` shim，无 shell 地 spawn `.cmd` 自 Node 的 CVE-2024-27980 缓解起抛 `EINVAL`，故经 `cmd.exe /c` 调用（不走 `shell: true`，那会 emit DEP0190）。
2. 走 `cmd.exe` 会被它重新解析命令行，所以参数里绝不能有 cmd 元字符 —— 由插件名正则与选项形态检查在入口挡掉。

### workspace 污染守卫

`runPnpm` 在 spawn 前从 cwd 起逐级上溯，若找到含 `pnpm-workspace.yaml` 的祖先，**直接抛错**：

> 在该处执行 pnpm 会改写 `<workspace>/pnpm-lock.yaml`，并可能把依赖实体落到那里。请把数据目录（`$CT_HOME`）放在任何 pnpm workspace 之外。

pnpm 以「上溯到的 workspace root」为 lockfile 与 store 的归属，与 cwd 无关；而 `$CT_HOME` 的默认建议位置与仓库同级，天然在 workspace 之外。

---

## 发布布局

`scripts/build-dist.mjs` 把 `bootstrap/` 编译到 `dist/`，其中命令注册表做一次改写：

| 源 | 产物 |
|---|---|
| `bootstrap/index.ts` | `dist/index.js`（`main`，启动层入口） |
| `bootstrap/cli/index.ts` | `dist/cli/index.js`（`bin: ct`） |
| `bootstrap/cli/commands/init.ts` | `dist/cli/commands/init.js` |
| `bootstrap/cli/commands/plugin.ts` | `dist/cli/commands/plugin.js` |
| `bootstrap/cli/commands/update.ts` | `dist/cli/commands/update.js` |
| `bootstrap/cli/commands.yml` | `dist/cli/commands.yml`（`file:` 由 `.ts` 改写为 `.js`） |
| `bootstrap/cli/kernel.json` | `dist/cli/kernel.json`（内核版本表，**`ct init` / `ct update` 读它**） |

**为什么命令实现必须是独立入口**：它们是运行时动态 `import()` 的目标，只打进 `cli/index.js` 是不够的 —— 这是本项目最容易漏的一步。

**为什么 `kernel.json` 要与 CLI 一起发布**：命令实现按**自己的位置**上溯一级去找它（源码布局 `bootstrap/cli/`、发布布局 `dist/cli/`），与 `commands.yml` 同一套定位方式。发布构建会解析一遍再写回 `dist/`，源表坏 JSON 在构建期就失败。

**构建后自检**：注册表里每一条 `file:` 都必须真实存在于产物中，否则构建失败。这是故意的红线：漏构建不会让构建报错，只会在消费者那里变成 `unfound command: <name>`，与「命令根本没注册」无法区分。

---

## 目录结构

```
bootstrap/cli/
├── index.ts               # 调度器（bin 指向它）
├── commands.yml           # bootstrap 级命令注册表
├── kernel.json            # 内核版本表（唯一真相源；与 vendor.mjs 共读）
├── actions/
│   ├── registry.ts        # 注册表读写 + 两级查找
│   ├── project.ts         # 仓库根锚定 / .env 读写 / CT_HOME 与 CT_PROFILE 解析 / 输入清洗
│   ├── pnpm.ts            # pnpm 调用 + workspace 守卫
│   ├── contract.ts        # 插件契约解析（PluginContract）
│   ├── kernel.ts          # 内核装配：读版本表 / 写内核依赖 / 差异比对
│   └── plugin.ts          # profileDir / ct.profile 读写 / 参数解析 / 命令表重放
└── commands/
    ├── init.ts            # ct init（建骨架 + 装配内核）
    ├── plugin.ts          # ct plugin 子命令分发
    └── update.ts          # ct update（升级内核副本）
```
