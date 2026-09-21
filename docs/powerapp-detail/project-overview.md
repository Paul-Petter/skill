# PowerApp 应用详情子应用（new-ui）项目梳理

## 一、项目概述

**new-ui**（仓库 `bk-powerapp-ui` 的 `new-ui/` 目录，包名 `power-app-ui`）是 PowerApp 应用管理平台前端的 **Vue 3 + TypeScript 全面重写版**，定位为聚焦「**应用详情**」场景的子应用，用于渐进式替换旧版 Vue 2 单页应用（`ui/`）中的对应能力。

子应用提供 K8s 应用（Application）的全生命周期管理界面：

- **应用操作**：同步 / 终止同步 / 差异（Diff）/ 同步策略 / 刷新（普通 / 强制）/ 重建 / 删除 / Pod 清零 / 重启工作负载；
- **资源观测**：资源树拓扑（G6）、资源表格、Manifest / YAML Diff（Monaco）、事件时间线、部署历史；
- **主机视图**：节点卡片 + 蜂窝布局 + 节点 / Pod 监控（CPU / MEM）+ 应用外 Pod 抽屉（YAML / 日志 / Web 终端 xterm）；
- **网络拓扑**：LB → Ingress → Service → Pod 链路推导（覆盖标准 K8s Ingress、BCS Ingress、PortPool、Istio、Gateway API）；
- **发布管理**：Rollout 发布单列表 / 详情 / 批量操作 / 中止与回滚；
- **AI 诊断**：嵌入式 AI 诊断面板（宿主注入蓝鲸 AI SDK，流式答复解析回填）。

**双形态构建**（`vite.config.mts:108-126`）：

- **独立 SPA**：`src/main.ts` 挂载 `#app-detail`，走 `DefaultLayout` 嵌套路由（hash / history 双模式）；
- **Library 产物**：Vite lib 模式打包为 `application.{format}.js` + `style.css`，入口 `src/pages/application/index.ts` 对外导出 `bkui / i18n / global / pinia / router` 模块、`ApplicationDetail` 组件、`createApp`、`showMessage`，供宿主页（旧版 Vue 2 平台）集成，实现新旧版本**共存与渐进式替换**；`js-yaml / monaco-editor / dayjs / diff` 四个依赖 externals 外置，由宿主全局提供以共享依赖。

其他能力：**多项目隔离**（`X-Project` 请求头，取自 URL path / localStorage）、**国际化**（vue-i18n 11 + `locales/` 下中英两份 yml）、**SSE 流式实时更新**（应用 watch / 资源树 / Pod 监控 / 事件流）。

规模：`src/` 下约 **218 个源文件**（**128 个 Vue 组件 + 79 个 TS 文件**），其中应用详情页 `src/pages/application/` 127 文件、组合式函数 **18 个**、API 接口定义 **78 个**（application 63 + rollout 8 + event-source 7）。

## 二、模块划分

路由入口 `/project/:project/application/:applicationName/:tabName`（`src/modules/router.ts:12-51`），Tab 级页面由 `tab-router.vue` 的异步组件映射表驱动（`tab-router.vue:21-50`，`defineAsyncComponent` 懒加载 + Suspense）：

| # | 模块（Tab） | 职责 |
|---|---|---|
| 1 | `summary` — 应用概要 | 应用基本信息、项目 / 应用组 / 同步策略展示 |
| 2 | `detail` — 应用详情（核心，含 `baseinfo/`、`components/` 76 文件、`table/`、`topo/` 资源树） | 基础信息栏、工具栏操作（同步 / 终止 / 差异 / 更多操作 / Pod 清零 / 重启）、资源树拓扑（`topo/app-topo.vue`）、资源表格（`table/app-table.vue`）、同步 / 删除 / 重建等对话框与抽屉 |
| 3 | `history` — 部署历史 | 历史版本列表、版本 Diff（`use-history-diff`）、回退 |
| 4 | `event` — 事件 | 应用事件时间线与聚合视图 |
| 5 | `hosts` — 主机视图（22 文件） | 节点卡片网格 + 蜂窝视图、节点 / Pod 监控、节点详情抽屉（YAML / Pod 列表）、应用外 Pod 抽屉（YAML / 日志 / Web 终端） |
| 6 | `network` — 网络拓扑 | LB → Ingress → Service → Pod 链路拓扑（复用 `topo/net-topo.vue` + `net-topo-builder.ts`） |
| 7 | `rollout` — 发布管理 | Rollout 发布单搜索 / 状态过滤 / 批量操作 / 详情步骤进度（SSE 实时刷新） |

支撑层：

| # | 支撑层 | 职责 |
|---|---|---|
| 8 | `src/api/` | 原生 fetch 封装（`ConsoleFetch` 类：URL `$变量` 模板替换、`X-Project` 注入、silent 静默模式）、拦截器 + AbortController、请求队列与取消、SSE 双方案（`event-source.ts`）、接口模块（application / rollout / event-source-api） |
| 9 | `src/stores/` | Pinia `application.ts`（841 行）：应用详情、资源树、集群节点、监控数据、Revision Metadata（按 SHA 缓存）、SSE 更新等全局状态 |
| 10 | `src/composables/`（18 个） | `use-ai-diagnosis`（AI 诊断会话管理）、`use-event-source`（SSE 重连管理）、`use-pod-metrics` / `use-node-pods-metrics`（监控自适应节流）、`use-editor-diff` / `use-history-diff`（Monaco Diff）、`use-rollout-history`、`use-resource-actions`、`use-visibility-change`、`use-interval` 等 |
| 11 | `src/components/`（33 文件） | 通用组件：editor-diff（Monaco Diff 封装）、resource-details 侧滑详情、event-view 事件视图、flex-row 等布局组件 |
| 12 | `src/modules/` | 自动安装模块（`bkui` / `global` / `head` / `i18n` / `pinia` / `router`），由 `main.ts` 通过 `import.meta.glob` eager 批量 install |
| 13 | `src/layouts/` + `src/pages/app/` | DefaultLayout 布局与 404 页 |

## 三、技术栈

### 框架层

- Vue **3.4.27**（Composition API + `<script setup>`）+ TypeScript **5.4.5** + vue-router **4.3.2**（嵌套路由，hash / history 双模式）+ Pinia **2.1.7**（替换旧版 Vuex）+ vue-i18n **11.3.0**（组合式 API）+ `@vueuse/core` 10.9

### UI 层

- `bkui-vue` **2.0.2-beta.70**（蓝鲸官方 Vue 3 组件库，锁定 beta 版本）+ `@blueking/tdesign-ui` 0.0.2-beta.13 + `@blueking/table` 0.0.1-beta.42（Vue 3 表格）
- `tippy.js` 6.3.7（tooltip）+ `vue3-toastify` 0.2.9（全局消息，经 `src/plugins/message.ts` 统一封装）

### 样式层

- **UnoCSS 66**（`mode: 'vue-scoped'`，presetUno / presetAttributify / presetTypography + transformerDirectives / transformerVariantGroup，`uno.config.ts`）
- less / scss，注入 `bk-prefix` 前缀变量（`vite.config.mts:91-98`），配合自研 PostCSS 插件 `plugins/css.prefix.js` 统一组件库样式前缀

### 构建层

- **Vite 6.4.2**：dev server 代理 `/api/v1`、`BK_` 环境变量前缀、`~` / `@` 路径别名
- **Library 模式**：入口 `src/pages/application/index.ts`，产物 `application.{format}.js` + `style.css`；externals 外置 `js-yaml / monaco-editor / dayjs / diff`（`vite.config.mts:108-126`）
- `unplugin-vue-components` 31（组件自动注册 + `components.d.ts` 类型生成）、`@intlify/unplugin-vue-i18n` 11（yml 语言包编译，runtimeOnly）
- monaco 5 类 worker（editor / json / css / html / ts）`?worker&inline` 预构建（`vite.config.mts:127-136`）

### 可视化 / 编辑器 / 终端

- `@antv/g6` **5.0.45**：资源树拓扑（`topo/app-topo.vue` + resource-node / polyline / minimap / context-menu / flow-line 等 13 个 ts 配套）与网络拓扑（`net-topo-builder.ts`，14 种节点类型链路推导）
- `echarts` 5.5.1：节点 / Pod 监控图表
- `monaco-editor` 0.51：YAML 查看与 Diff（`src/components/editor-diff/`）
- `@xterm/xterm` 5.5 + fit / search / web-links addons：应用外 Pod Web 终端（`hosts/foreign-pod-terminal.vue`）

### AI 能力

- 嵌入式 AI 诊断面板：宿主注入 `getAIChatHelper()` 获取蓝鲸 AI SDK ChatHelper，会话管理 + 流式答复按 H1 切分解析（`src/composables/use-ai-diagnosis.ts`）
- 输入安全：`src/common/ai-input-validation.ts`（`isSendableAIPrompt` / `sanitizeAIPrompt`），并兼容旧版 bus 事件透传

### 流式通信（SSE）

- `event-source-polyfill` 1.0.31（GET 方案，自定义 `X-Project` 头，`src/api/event-source.ts:20-37`）
- `sse.js` 2.6.0（POST 方案，payload JSON 序列化，`src/api/event-source.ts:39-62`）
- `use-event-source.ts` 统一管理：断线自动重连（最大 3 次、5s 间隔）、beforeOpen 检查、错误关闭策略

### 工具库

- **原生 fetch**（无 axios）：`ConsoleFetch` 类封装（URL `$变量` 模板、GET 参数 / body 分流、formData 支持）
- `dayjs` 1.11（relativeTime / duration 插件 + zh-cn locale）、`diff` 8.0.2（文本 Diff）、`json-merge-patch`（SSE 增量合并）、`js-yaml` 4.1、`lodash-es`、`uuid` 11、`vue-clipboard3`、`query-string`

### 测试与工程化

- `vitest` 3.2.4 + jsdom + `@vue/test-utils`（`test/` 下组件测试）；`vue-tsc` 类型检查；`@blueking/eslint-config-bk` + `eslint-plugin-simple-import-sort`
- **pnpm 10.33.4**（`pnpm.overrides` 统一传递依赖版本）+ `taze` 依赖升级 + `vite-bundle-visualizer` 产物分析

### 服务与部署

- `Dockerfile`：`node:20-alpine` 多阶段构建，corepack + pnpm store 缓存挂载，`http-server` 静态服务（`pnpm run server`）
- 依赖外置约定：lib 产物与宿主共享 `js-yaml / monaco-editor / dayjs / diff` 全局变量
