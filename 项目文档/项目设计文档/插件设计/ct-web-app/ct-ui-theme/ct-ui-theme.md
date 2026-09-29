# ct-ui-theme

状态：🟢 已实现（`0.0.1-alpha.1`；v1 样式表注入流程；见 `cordis-tavern-plugin/ct-ui-theme/`）

主要目的：构建一个统一的ui token库通过服务暴露，让全局主题相同。
主要调用服务
node侧cordis
依赖persist.json对主题文件进行持久化
依赖webServer服务让插件在前端后端通信，支持持久化后的主题文件传输

浏览器侧cordis：
提供服务theme
v1版本暂时不提供任何修改主题注册主题的方法，先把构建样式表，react组件全局引用成功流程跑通

构建样式表步骤
1. 使用CSSStyleSheet()创建空全局样式表
2. 使用 `replaceSync()` 方法向样式表内填充css变量引用规则和变量定义。这是现代规则中常用的方法
3. 将样式表“采用”到文档中
   这是最关键的一步。**不把样式表插入 DOM 节点**，而是将它赋值给 `document.adoptedStyleSheets` 或 `ShadowRoot.adoptedStyleSheets`
，
测试方法
由于目前仅设置了注入方法，变量命名规则以及css文件未提供，所以测试需要插件ct-ui-layout协助
测试为定义一个css变量
--ct-red:#FF0000
并通过
.red-test
 color:var（--ct-red）
 这类react组件引用css文件内变量的通用方法进行构建样式表
同时修改ct-ui-layout，让他目前的测试字体颜色变为引用颜色