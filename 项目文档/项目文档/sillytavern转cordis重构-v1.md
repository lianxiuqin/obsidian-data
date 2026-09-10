整理后版本如下：

# 参考架构

[[sillytavern]] 是一个本地安装、开源、专为“高级用户”打造的 LLM 前端界面。它本身不提供模型，而是作为一个功能极其强大的“控制中枢”，让用户能够精细驾驭各类 AI 模型，进行深度角色扮演、故事创作和互动对话。

[[cordis]] 是一个元框架（Meta Framework），即“用于构建框架的框架”。它不耦合任何具体业务领域，只专注于解决一个核心问题：如何让软件各组件安全地组合、解绑与重组。

# 重构目的

旨在大幅增强 SillyTavern 的扩展能力，并通过内置插件包实现 SillyTavern 的原功能。

# 项目设计

## 核心基建层

0. **启动层**：实现 plugin / plugin config 的多层级覆盖，并通过 CLI 命令工具进行插件管理；它是配置与 CLI 等技术层面的复合层。

1. **自定义构建工具（Build Tool）**：基于 esbuild 封装，根据包内预设的库（如 React、Vite 等）避免构建时重复打包，防止项目体积无谓增大。

2. **插件注册表 / 环境表**：通过自动维护 `package.json` 中的声明，完成环境构建与维护。

## 插件契约层

3. **核心声明**：开发者主动在 `package.json` 中维护约定字段 `st`，用于声明和发现。

   ```json
   {
     "st": {
       "bundle": {
         "path": "./cordis.patch.yml"
       }
     }
   }
   ```

4. **前端 / 前后端插件声明**：通过 `st.client` 声明。该字段由插件解析；由于涉及执行操作，因此与 Cordis 无关。

   ```json
   {
     "st": {
       "client": {
         "platform": "web",
         "inject": ["plugin-name"],
         "immediately": true,
         "external": ["base-module"]
       }
     }
   }
   ```

   - `platform`：声明该插件包含前端结构。
   - `inject`：声明该插件的前端结构依赖哪些插件。
   - `immediately`：声明该插件是预加载，还是仅注册工厂函数以进行惰性加载。
   - `external`：声明该插件所需的包，在**自定义构建工具**中生效。

5. **命令声明**：通过 `st.commands` 字段声明。

   ```json
   {
     "st": {
       "commands": [
         {
           "command": "name",
           "path": "./……/name.ts",
           "explain": "",
           "subcommands": [
             {
               "subcommand": "",
               "explain": ""
             }
           ]
         },
         {
           "command": "name2",
           "path": "./……/name2.ts",
           "explain": "",
           "subcommands": []
         }
       ]
     }
   }
   ```

   - `commands`：声明多个命令。
   - `command`：声明一个命令名。
   - `explain`：说明命令功能。
   - `subcommands`：声明子命令组。
   - `subcommand`：声明单个子命令。

### `package.json` 声明汇总

```json
{
  "st": {
    "bundle": {
      "path": "./cordis.patch.yml"
    },
    "client": {
      "platform": "web",
      "inject": ["plugin-name"],
      "immediately": true,
      "external": ["base-module"]
    },
    "commands": [
      {
        "command": "name",
        "path": "./……/name.ts",
        "explain": "",
        "subcommands": [
          {
            "subcommand": "",
            "explain": ""
          }
        ]
      },
      {
        "command": "name2",
        "path": "./……/name2.ts",
        "explain": "",
        "subcommands": []
      }
    ]
  }
}
```