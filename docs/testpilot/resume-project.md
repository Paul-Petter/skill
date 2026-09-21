# 简历项目经历（bk-testpilot）

> 以前端视角撰写；内容均可在仓库中找到依据，未虚构与夸大。「时间」「角色」为占位项，请自行替换。

## 项目经历

**项目名称**：bk-testpilot —— 蓝鲸测试流程管理平台（Web 前端）

**时间**：20XX.XX – 至今（请自行替换）

**角色**：前端开发工程师（请自行替换）

**技术栈**：Vue 3 / TypeScript / Vite / bkui-vue / UnoCSS / Pinia / vue-router / vue-i18n / ECharts / OpenAPI / Docker

**项目描述**：

面向测试团队的流程管理平台，打通「TAPD 需求 / Bug → 测试用例与脚本 → 测试计划执行 → 结果上报 → 测试报告」完整链路。后端为 Go + Gin + RabbitMQ + Kubernetes 沙箱执行；前端（Vue 3 + Vite + bkui-vue）承载用例管理、环境管理、计划编排、执行报告与数据看板等控制台界面，支持多空间隔离与中英双语。

**主要职责与成果**：

- **API 工程化**：基于 OpenAPI swagger 搭建前端 API 客户端与 TypeScript 类型自动生成方案（`pnpm gen:api:v1`，覆盖 platform / suit / env / plan / ai 五个业务域），实现前后端契约同步与类型安全，消除手写接口层的漂移问题
- **基础设施封装**：实现统一 HTTP 客户端 `HttpFetch`——401 跳转登录 / 403 无权限 / 400 错误详情的全局拦截、URL `{var}` 占位符替换、数组参数 `comma / exploded` 双序列化，业务侧免写重复异常处理
- **多环境部署方案**：设计 `__BTP_RT_BK_*__` 运行时占位符机制（开发期 Vite 插件替换、生产期容器启动注入），一份构建产物适配多环境；处理多租户 `BK_USER_URL` 占位替换与开发环境 CORS 反向代理
- **设计系统落地**：建立 Design Tokens 三级链路（Figma 色板 → `--bk-tp-*` CSS 变量 → UnoCSS 原子类 / shortcut 映射），全站禁用硬编码色值，保障视觉一致性与可维护性
- **业务模块开发**：完成测试用例集（含 TAPD 单据关联与 AI 推荐上下文）、测试环境、测试计划、测试报告、视图看板、空间设置（基础信息 / 成员管理 / 审计日志）、个人 Token 等八个一级页面模块（`src/pages/` 共 104 个文件）
- **通用能力沉淀**：沉淀 32 个通用组件（人员选择器、表格异常态、单元格链接等）与 20 个 composables（搜索过滤、表格高度自适应、轮询、离开确认等），组件经 unplugin-vue-components 自动按需注册
- **架构细节**：多空间路由守卫（空间合法性校验 + 切换时强制视图刷新）、二级布局系统（`vite-plugin-vue-layouts` + 路由 meta 约定）、Pinia 持久化白名单（localStorage 插件）、YAML 词典国际化（placeholder 文案统一生成）
- **工程化**：ESLint / Prettier / Stylelint + husky / lint-staged（含暂存区类型检查）+ vue-tsc 严格类型检查 + Docker 化部署（nginx + 运行时变量注入）

**量化口径说明**（写入简历前请按实际情况核对）：

- 页面（104）/ 组件（32）/ composables（20）为当前代码快照的文件统计，非业务指标
- 如需补充性能优化、测试覆盖、上线规模等数据，请以真实数据为准再写入

## 精简版（一段式，适合简历空间紧张时使用）

**bk-testpilot —— 蓝鲸测试流程管理平台（前端）**｜20XX.XX – 至今｜Vue 3 / TypeScript / Vite / bkui-vue / UnoCSS / Pinia

参与测试流程管理平台前端开发：基于 OpenAPI 搭建 API 客户端与类型自动生成方案，封装统一 HTTP 拦截层与运行时多环境变量注入机制，落地 Design Tokens 设计系统；完成测试用例集、测试环境、测试计划、测试报告、数据看板等八个业务模块，沉淀 32 个通用组件与 20 个 composables；配套 husky / lint-staged / vue-tsc 工程化体系与 Docker 部署方案。
