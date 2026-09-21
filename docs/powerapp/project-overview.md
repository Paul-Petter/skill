# PowerApp 项目梳理

## 一、项目概述

**PowerApp**（仓库 `bk-powerapp-ui`）是部署于**蓝鲸 PaaS** 的前后端分离单页应用（SPA），定位为 **Kubernetes + Terraform 混合式应用管理平台前端**（GitOps 风格）。

平台同时管理两类资源：

- **K8s 应用层**：Application / ApplicationSet 的部署、同步（Sync）、Diff、批量操作，以及底层 K8s 资源（Events / Logs / Manifest / 重启）的直接管理；
- **Terraform 基础设施层**：Layer / LayerSet 的 `init → plan → apply → policy` 全生命周期管理与 State 管理（state-rm / unlock / refresh）。

平台支持**多项目隔离**（通过 `X-Project` 请求头，取自 URL path / localStorage），内置 **AI 助手**（`@blueking/ai-blueking`）与 **MCP 工具权限**管理，并提供 Operator 应用市场、部署报告、审计等配套能力。

规模：`src/` 下约 **290+ 源文件**、**193 个 Vue 组件**。

## 二、模块划分

依据 `src/views/` 目录与 `src/router/index.js`，共 **11 大功能模块**：

| # | 模块（路由） | 职责 |
|---|---|---|
| 1 | `projects` — 项目管理 | 项目列表与进入，多项目隔离入口 |
| 2 | `applications` — 应用管理（36 文件） | K8s 应用的同步 / 批量同步 / 管理资源 / Manifest Diff 等 |
| 3 | `applicationsets` — 应用集管理 | ApplicationSet 列表与详情，批量拉起应用 |
| 4 | `layer` / `layersets` — 基础设施层 | Terraform Layer 的 init / plan / apply / policy / state 管理 / AI 分析 / 批量操作 / 事件流实时输出 |
| 5 | `resources` — 资源管理（25 文件） | K8s 资源的 events / logs / restart / manifest / yaml diff |
| 6 | `markets` — 应用市场 | Operator 市场浏览与详情、一键部署 |
| 7 | `secrets` — 密钥管理 | 密钥列表与详情 |
| 8 | `reports` — 部署报告 | 部署前报告 / 部署报告及详情 |
| 9 | `settings` — 设置 | 7 个子页：repo（代码库）/ charts / cluster（集群）/ notice（通知）/ po / audit / **mcp** |
| 10 | `permission` — 权限管理 | 用户 / 用户组权限配置 |
| 11 | `audit` — 审计 | 操作审计记录 |

## 三、技术栈

### 框架层

- Vue 2.7.0（组合式 API 可用的 Vue 2 终版）+ Vuex 3.1.1 + Vue Router 3.0.6 + vue-i18n 8（国际化）

### UI 层

- `bk-magic-vue` 2.5.10（蓝鲸官方组件库），经 `babel-plugin-import-bk-magic-vue` 按需加载
- `tippy.js`（tooltip）

### 构建层

- `@blueking/cli-service-webpack` 0.0.7（蓝鲸定制 CLI）+ Webpack + PostCSS（mixins / nested / vars / preset-env）
- 自研插件：`replace-static-url-plugin.js`（`{{ .BK_STATIC_URL }}` 模板替换，适配蓝鲸 PaaS）、`build-version.js`（生成 `static/verify.json` 版本校验）
- `env.js` 多环境 `AJAX_URL_PREFIX` 配置

### 编辑器

- `monaco-editor` 0.51.0，经 `MonacoWebpackPlugin` 双维度裁剪（仅保留 `yaml` 语言、剔除 9 个 features）——见 `bk.config.js:104-120`

### AI 能力

- `@blueking/ai-blueking` 2.1.4-beta.4（Vue2 版 AI 助手组件）
- MCP 工具权限管理（`src/api/mcp.js`）
- 输入校验（`src/common/ai-input-validation.js`）

### 流式通信（SSE）

- `event-source-polyfill` 1.0.31（GET 方案，`src/api/event-source.js`）
- `sse.js` 2.6.0（POST 方案，`src/common/sselib.js`，支持自定义 headers + payload，5 分钟心跳超时）

### 工具库

- `axios`（HTTP，带路由切换取消队列）、`dayjs` 1.11.18、`lodash-es`、`diff` 5.1（YAML Diff）、`marked` 4.3 + `github-markdown-css` + `xss`（安全的 Markdown 渲染）、`@e965/xlsx`（导出）、`query-string`、`uuid`

### 服务与部署

- `express`：`mock-server/`（开发联调）+ `paas-server/`（可部署的 Node 静态服务）
