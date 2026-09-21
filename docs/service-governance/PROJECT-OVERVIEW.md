# bkms-ui 项目梳理

> 本文按「概述、模块、技术栈」三部分整理，内容取自当前代码仓库（`package.json`、`README.md`、`docs/` 及 `src/` 源码），文件数量为仓库当前统计。

## 一、概述

`bkms-ui` 是蓝鲸智云服务治理平台（BlueKing Service Governance）的 Web 前端，基于 Vue 3、TypeScript、Vite、Vue Router 和 Pinia 构建（见 `README.md`）。平台面向微服务全生命周期治理，前端提供从应用构建、制品管理、部署发布到可观测（监控、告警、仪表盘、资源拓扑）的完整操作界面，支持 tRPC / TAF / Helm / Agones 四类应用类型（`src/modules/router.ts`、`src/config/navigation/app.ts`）。

**规模一览**：

| 维度 | 数据 |
| --- | --- |
| `src/` 文件总数 | 668 个（339 个 `.vue` + 247 个 `.ts`） |
| 业务页面模块 | 8 个（`src/pages/`） |
| 跨页面复用组件 | 91 个文件（`src/components/`，其中 86 个 `.vue`） |
| 可复用 composables | 45 个（`src/composables/`） |
| Pinia Store | 11 个（`src/stores/`） |
| API 层 | 53 个 TS 文件（`src/api/`，其中 42 个由 Swagger 自动生成） |
| 自动生成类型 | 76 个（`src/@types/v1/`） |
| E2E 测试 | 78 个文件（含 17 个 Gherkin `.feature` 用例，`e2e/`） |
| 单元测试 | 9 个测试文件（`test/`） |
| 国际化 | zh-CN / en-US 双语言（`locales/*.yml`） |

工程配套：Docker 多阶段构建（`node:23-alpine` 构建 + `nginx:1.30-alpine` 运行）实现「一次构建、多环境部署」；ESLint / Stylelint / Prettier / Biome + husky + lint-staged + vue-tsc 保障代码质量。

## 二、模块

### 2.1 业务模块（`src/pages/`）

| 模块 | 职责 | 主要内容 |
| --- | --- | --- |
| `home` | 工作台与空间选择 | 空间列表、空间切换、团队空间、空态页 |
| `application` | 应用管理（最大模块，207 个文件） | 应用创建（tRPC / TAF / Helm / Agones 模板）、应用列表；应用详情导航按应用类型分两套（`app.ts`）：构建管理、制品管理、部署管理（tRPC / Helm）、观测数据、仪表盘、监控告警、北极星、网络访问、组件配置、应用配置、基本信息、应用编排（Helm 类）、操作记录、K8s 资源拓扑 |
| `env` | 部署环境管理（33 个文件） | 环境列表 / 创建 / 删除、环境详情（基本信息、环境配置、观测数据）、集群组件（端口池）、集群健康诊断、公共环境变量、泳道配置、APM 实例 |
| `basic` | 空间设置（14 个文件） | 空间基本信息、操作记录、应用默认配置 |
| `marketplace` | 组件市场（12 个文件） | 组件管理、组件输入 / 输出配置（含 Monaco Editor 模板编辑） |
| `platform` | 平台管理 | 空间管理（含空间详情）、平台管理员 |
| `plugin` | 插件配置 | 组件配置列表 |
| `app` | 全局页面 | 403 无权限页、404 未找到页、全局 footer |

### 2.2 支撑层（`src/`）

| 目录 | 职责 |
| --- | --- |
| `api/` | 网络基础设施：`ConsoleFetch` 封装（`fetch.ts`）、拦截器（`interceptors.ts`）、请求队列（`request-queue.ts`）、链路追踪（`trace-id.ts`）、客户端（`clients.ts`）；`modules/` 按领域暴露接口，其中 `modules/v1/`（42 个文件）由 Swagger 自动生成 |
| `@types/v1/` | 自动生成的 v1 API 类型（76 个文件），由 `pnpm gen:api:v1` 生成，禁止手工修改 |
| `components/` | 跨页面复用组件：Monaco Editor 封装（`monaco-editor/`）、Web 终端（`terminal.vue`）、Grafana 仪表盘 / 监控 iframe、Git / 流水线 / 集群 / 成员选择器、Markdown 渲染、骨架屏等 |
| `composables/` | 可复用组合式逻辑（45 个），如页面状态与 URL query 双向同步（`use-url-query-sync.ts`） |
| `stores/` | Pinia Store（11 个）：user、space、apps、application、app-detail、deploy-env、env-detail、env-list-state、platform-config、trpc-deploy、ui |
| `modules/` | 应用级模块（bkui / global / i18n / pinia / router），实现 `install` 方法，启动时经 `import.meta.glob` 自动安装 |
| `layouts/` | 通用页面布局，配合 vite-plugin-vue-layouts 按 `meta.layout` 装配 |
| `config/navigation/` | 静态导航配置（app / basic / env / platform / plugin） |
| `directives/`、`common/`、`types/` | 可复用 DOM 指令、通用常量枚举、跨模块类型契约 |
| `assets/`、`fonts/`、`styles/` | 静态资源、图标字体、全局样式 |

### 2.3 启动链路与依赖方向

启动顺序（`src/main.ts`）：注册 ECharts 渲染器与全局样式 → `getUser` 获取用户信息 → 创建 Vue 应用 → `import.meta.glob` 自动安装 `modules` → 注册监控图表桥接 → 挂载 `App.vue` → 用户信息写入 `user` Store。

推荐依赖方向（`docs/ARCHITECTURE.md`）：

```text
pages / layouts
    ↓
components / composables / stores
    ↓
api
    ↓
后端或外部服务
```

## 三、技术栈

版本与 `package.json` 逐字对应。

### 3.1 核心框架

| 依赖 | 版本 | 用途 |
| --- | --- | --- |
| vue | ^3.5.16 | 应用框架 |
| typescript | ^5.4.5 | 类型系统 |
| vite | ^6.4.2 | 构建工具 |
| pinia | ^3.0.3 | 状态管理 |
| vue-router | ^4.5.1 | 路由（Hash History，基础路径由 `BK_SITE_URL` 控制） |
| vue-i18n | ^11.1.5 | 国际化（zh-CN / en-US） |

### 3.2 UI 与蓝鲸生态

| 依赖 | 版本 |
| --- | --- |
| bkui-vue | 2.0.2-beta.97 |
| @blueking/table | 0.0.1-beta.42 |
| @blueking/ediatable | 0.0.1-beta.32 |
| @blueking/task-log | 0.0.14 |
| @blueking/monitor-vue3-components | 1.0.7-beta.1 |
| @blueking/bk-user-selector | ^0.1.9 |
| @blueking/date-picker | ^3.0.7 |
| @blueking/notice-component | ^2.0.7 |
| @blueking/platform-config | ^1.0.5 |
| @blueking/xss-filter | ^0.0.5 |
| unocss / less | ^66.5.4 / ^4.2.0 |

### 3.3 可视化与编辑器

| 依赖 | 版本 | 用途 |
| --- | --- | --- |
| @antv/g6 | 5.0.50 | K8s 资源拓扑图引擎 |
| g6-extension-vue | ^0.1.0 | Vue 组件作为 G6 自定义节点 |
| echarts | ^5.6.0 | 图表（应用启动时全局注册渲染器与组件，与监控组件共享实例） |
| monaco-editor | ^0.52.2 | 代码 / YAML 编辑器（全仓 21 个文件使用） |
| marked + marked-highlight + highlight.js | ^15.0.7 / ^2.2.1 / 11.5.0 | Markdown 渲染与高亮 |
| vue-virtual-scroller | 2.0.0-beta.8 | 长列表虚拟滚动 |

### 3.4 工具库

@vueuse/core ^10.9.0、dayjs ^1.11.13、lodash-es ^4.18.1、js-yaml ^4.1.0、tippy.js ^6.3.7、pluralize ^8.0.0、figlet ^1.8.1。

### 3.5 工程化与质量

| 类别 | 工具 |
| --- | --- |
| 包管理 | pnpm 10.33.4（含依赖安全 overrides：brace-expansion、qs、minimatch、koa 等） |
| 代码规范 | ESLint、Stylelint、Prettier ^3.9.6、Biome、@blueking/bkui-lint 0.1.8 |
| 提交钩子 | husky ^9.1.7 + lint-staged ^15.2.2 |
| 类型检查 | vue-tsc ^2.0.19（全量 `typecheck` + 增量 `typecheck:staged`） |
| Vite 插件 | unplugin-vue-components（组件自动注册）、vite-plugin-vue-layouts、vite-svg-loader、vite-plugin-compression、vite-plugin-vue-devtools、@intlify/unplugin-vue-i18n |
| 单元测试 | Vitest ^3.0.9 + @vue/test-utils ^2.4.6 + jsdom ^26.1.0 |
| E2E 测试 | Playwright + playwright-bdd（Gherkin），17 个 `.feature` 用例，容器化运行（node:22 + Chromium） |

### 3.6 部署

| 项 | 说明 |
| --- | --- |
| 镜像 | `Dockerfile.prod` 多阶段构建：`node:23-alpine` 构建 + `nginx:1.30-alpine` 运行 |
| 运行时配置 | 构建产物保留 `__BKMS_RT_BK_XXX__` 占位符，`docker-entrypoint.sh` 容器启动时替换（一次构建、多环境部署） |
| 端口 / 探活 | 5000 端口，镜像内置 HEALTHCHECK |
| Nginx 策略 | SPA 回退、`/assets/` 长缓存 immutable、HTML no-cache、nosniff / frame / Referrer-Policy 等安全头 |
