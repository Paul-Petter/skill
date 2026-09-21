# bk-testpilot 项目梳理

> 基于 2026-09 代码快照整理；文中「规模」类数字均为文件统计口径，非业务运行指标。

## 一、项目概述

**bk-testpilot** 是蓝鲸智云生态下的测试流程管理平台，整体由三部分组成：

| 组成 | 技术形态 | 说明 |
| --- | --- | --- |
| 后端服务 | Go 1.25 + Gin | API、测试计划执行编排、Worker 与沙箱 Agent 调度 |
| 前端 | Vue 3 + Vite + bkui-vue | 仓库 `ui/` 目录 |
| 命令行工具 | `btp-cli` | 通过「个人 Token」访问平台 API |

**核心业务主链路**：

```text
TAPD Story/Bug → 用例与脚本 → 计划执行 → 结果上报 → 测试报告
```

**整体架构采用控制面 / 数据面分离**：

- **控制面**（API / Plan）：管理权限、快照、状态与报告。`plan` 是任务状态与 Session 收敛的唯一写权威，负责组合 `suit`（用例）与 `env`（环境）。
- **数据面**（MQ / Worker / Kubernetes 沙箱 Agent）：`plan` 通过 RabbitMQ 下发 `dispatch / abort / retry` 命令；Worker 只负责创建 / 删除 / 重建执行环境（Pod 生命周期），不写业务状态；沙箱 Agent 通过 Runtime API 拉取任务、执行用例、上报心跳与结果。

**关键设计约束**：执行历史只读取 run / session 冻结快照，报告不得回读后续变化的用例、数据集或环境；Token、Cookie、凭证等敏感值不得进入日志。

（依据：仓库根 `AGENTS.md`、`docs/development/services/test-plan-execution.md`）

## 二、功能模块

### 2.1 前端页面模块

依据 `src/modules/router.ts` 路由表（一级导航 → 二级侧栏）：

| 一级导航 | 二级模块 | 路由 | 功能 |
| --- | --- | --- | --- |
| 首页 | — | `/home` | 空间首页 |
| 测试管理 | 测试用例集 | `/:space/cases` | 用例集 / 用例 CRUD 与详情、关联 TAPD 单据（`tapd-picker-slider`）、AI 推荐上下文 |
| 测试管理 | 测试环境 | `/:space/env` | 环境列表与详情，配置被测系统访问入口与执行机资源 |
| 测试管理 | 测试计划 | `/:space/plans` | 计划列表 / 新建 / 编辑 / 详情，配置执行环境、用例集与快照执行历史 |
| 测试管理 | 测试报告 | `/:space/reports` | 查看测试执行报告 |
| 视图看板 | — | `/:space/dashboard` | 空间测试数据概览（ECharts） |
| 空间设置 | 基础信息 / 成员管理 / 审计日志 | `/:space/settings/*` | 空间资料、成员与权限、操作审计 |
| 个人设置 | 个人 Token | `/profile/token` | 供 `btp-cli` 等命令行工具访问平台 API |
| （预留） | 测试数据 | `/:space/data` | 已开发但路由暂时注释隐藏（「测试数据暂时不开发，暂时隐藏」） |

### 2.2 后端业务域

| 模块（`internal/modules/`） | 职责 |
| --- | --- |
| `platform` | 空间、成员、审计日志等平台基础能力 |
| `suit` | 用例集 / 用例管理 |
| `env` | 测试环境管理 |
| `plan` | 测试计划编排、执行状态聚合、报告生成（组合 `suit` 与 `env`） |
| `ai` | AI 能力（如推荐上下文） |

其他后端目录：`cmd/`（API / Worker / Agent / CLI 入口）、`internal/infra/`（DB、MQ、K8s 等适配器）、`internal/worker/`（命令消费与环境编排）、`pkg/`（auth、HTTP、审计等跨模块能力）、`api/openapi/v1/`（OpenAPI 契约唯一源）。

### 2.3 前端通用能力层

| 层 | 位置 | 规模 | 说明 |
| --- | --- | --- | --- |
| API 客户端 | `src/api/modules/` | 5 个业务域 | OpenAPI swagger 生成（`platform` / `suit` / `env` / `plan` / `ai`），禁止手改 |
| API 类型 | `src/@types/` | — | 同上，生成物 |
| 通用组件 | `src/components/` | 32 个文件 | `user-selector`、`table-exception`、`cell-link`、`toggle-card`、`resource-select-dialog` 等，`unplugin-vue-components` 自动注册 |
| Composables | `src/composables/` | 20 个 | 搜索 / 过滤 / 分页、表格空态与高度、轮询（`use-interval`）、离开确认、表单错误聚焦等 |
| 全局状态 | `src/stores/` | 3 个 | `app` / `space` / `user`；自研 localStorage 持久化插件（白名单：`user` / `space` / `deploy-env`） |
| HTTP 基础设施 | `src/api/` | — | `fetch.ts` 统一拦截、`clients.ts` 实例、`interceptors.ts` 底层封装 |
| 布局 | `src/layouts/` | 4 个 | `default` / `main` / `space-side` / `space` |

## 三、技术栈

### 3.1 前端（`ui/`）

| 项 | 选型 |
| --- | --- |
| 框架 | Vue 3.5 + TypeScript 5.7 + `<script setup>` |
| 构建 | Vite 6 + pnpm 10 |
| 路由 | vue-router 4（hash 模式）+ `vite-plugin-vue-layouts`（默认 `space-side` 布局） |
| 状态 | Pinia 2 + 自研 localStorage 持久化插件 |
| UI 组件 | bkui-vue 3.0.2（全局注册）；`@blueking/table`、`@blueking/bk-user-selector`、`@blueking/notice-component`、`@blueking/login-modal`、`@blueking/release-note` 等业务组件 |
| 样式 | UnoCSS + Design Tokens（`src/styles/tokens/`，`--bk-tp-*` CSS 变量 → `uno.config.ts` 原子类映射） |
| 国际化 | vue-i18n 11（YAML 语言包 `zh-CN` / `en-US`，`@intlify/unplugin-vue-i18n` 构建期编译） |
| 图表 | ECharts 6 |
| 长列表 | vue-virtual-scroller |
| 工具库 | lodash-es、dayjs、tippy.js、@vueuse/core |

### 3.2 后端

| 项 | 选型 |
| --- | --- |
| 语言 / 框架 | Go 1.25 + Gin |
| 消息队列 | RabbitMQ（Plan → Worker 下行命令 `dispatch / abort / retry`） |
| 执行环境 | Kubernetes（Worker 管理 Pod 生命周期，沙箱 Agent 执行用例） |
| API 契约 | OpenAPI v3（`api/openapi/v1/spec/*.yaml` 唯一源，`make gen` 生成代码） |

### 3.3 工程化与质量

| 项 | 选型 |
| --- | --- |
| 代码规范 | ESLint 9 + Prettier + Stylelint（含 `@blueking/bkui-lint`） |
| 提交钩子 | husky + lint-staged（含 staged 类型检查 `typecheck:staged`） |
| 类型检查 | vue-tsc（`pnpm typecheck`，构建前置） |
| API 同步 | 后端 `make gen` → 前端 `pnpm gen:api:v1` |
| 部署 | Docker（nginx + `docker-entrypoint.sh` 运行时环境变量注入），`Dockerfile.prod` |
| 后端测试 | `make test`（单元）/ `make test-integration`（集成，需 Docker） |
