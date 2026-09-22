---
tags:
  - obsidian
  - 插件开发
  - github
---

# Obsidian GitHub 插件开发

## 官方教程文档

[[Obsidian]] 官方提供了明确且完整的**插件开发**示例与文档。

### 官方文档

[docs.obsidian.md](https://docs.obsidian.md/) 的 **Plugins** 部分收录了以下内容：

- **Build a plugin**：手把手入门教程
- **Quickstart**：快速构建第一个插件
- **Anatomy of a plugin**：讲解插件结构与[[生命周期]]
- 编辑器扩展、视图插件、设置、事件等专题指南

### 官方示例模板

`https://github.com/obsidianmd/obsidian-sample-plugin`

该模板包含 [[TypeScript]] 配置、[[esbuild]] 构建脚本和基础 `main.ts` 示例，适合直接克隆或作为开发起点。

### API 参考与类型定义

`https://github.com/obsidianmd/obsidian-api`

提供 `obsidian.d.ts` **类型定义**，官方文档中也包含核心 [[API]] 说明与代码示例。

### 实践建议

- 使用独立的 [[Vault]] 开发插件，避免影响日常工作库。
- 遇到问题可前往 Obsidian 官方论坛的 **Developers & API** 板块求助。

**入门路径**：克隆官方示例插件 → 按 Quickstart / Build a plugin 跑通 → 查阅 API 文档与类型定义进行扩展。

## 项目开发文档

### 功能预想

在菜单按钮中加入 **push / pull** 按钮，方便与 [[GitHub]] 仓库保持同步。

### 具体开发

- 同步工具：[[GitHub API]]。
- 用户点击 push/pull 按钮后，插件通过 GitHub API 与**细粒度 Token**（Fine-grained Token）对仓库执行 [[push]] / [[pull]] 操作。

### 配置页面

配置页面应包含：

- **Token 输入栏**：用于输入细粒度 Token（Fine-grained Token）。
- **上传信息输入栏**：暂定三个字段，分别为姓名、邮箱和描述。
- **默认值**：描述输入栏提供默认值，暂定为 `{{YYYY-MM-DD}}` [[宏变量]]；不支持其他变量，也不支持正则替换。

### 工作流程

1. 用户点击按钮后，弹出二次确认对话框，展示本次需要更新/新建的文件数量与删除的文件数量，并请求用户确认是否执行 push/pull。
2. 用户确认后开始执行；同时，Obsidian 通知会显示自定义动画与提示语，实时反馈 push/pull 的进行中、成功与失败状态。

## 总结

本文档汇总了 Obsidian 官方插件开发的文档、示例模板与 API 类型定义，梳理了从入门到扩展的完整学习路径。在此基础上规划了一个 GitHub 仓库同步插件，涵盖细粒度 Token 配置、push/pull 操作与二次确认流程。配置页面支持默认宏变量，通知系统可实时反馈同步的进行中、成功与失败状态。