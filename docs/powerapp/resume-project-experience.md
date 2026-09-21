# 简历项目经历：PowerApp

> 说明：时间、角色、团队规模等占位项以 `【】` 标注，请自行替换；所有量化数据均可在代码库中佐证，未夸大。

---

**项目名称**：PowerApp —— Kubernetes + Terraform 混合式应用管理平台（前端）

**项目时间**：【20XX.XX – 至今，请自行替换】

**技术栈**：Vue 2.7 / Vuex / Vue Router / bk-magic-vue（蓝鲸组件库）/ Webpack（bk-cli-service-webpack）/ Monaco Editor / SSE（sse.js + event-source-polyfill）/ Axios / i18n / 蓝鲸 PaaS

**项目描述**：

PowerApp 是部署于蓝鲸 PaaS 的前后端分离单页应用，面向混合云场景提供 GitOps 风格的应用管理能力：覆盖 K8s 应用与应用集（Application/ApplicationSet）的部署、同步与 Diff、K8s 资源（Events/Logs/Manifest）管理、Terraform 基础设施层（Layer/LayerSet）的 init → plan → apply → policy 全生命周期与 State 管理，并集成 Operator 应用市场、部署报告、审计、权限、AI 助手（蓝鲸 AI 组件 + MCP 工具）等能力。前端包含 **11 个功能模块、约 290+ 源文件、193 个 Vue 组件**，支持多项目隔离与国际化。

**主要职责**（按实际情况增删）：

- 负责核心业务模块的前端开发与迭代，包括应用管理、Terraform Layer 生命周期（多阶段状态机 + 事件流实时输出 + 批量操作）与 K8s 资源管理；
- 实现双方案 SSE 流式通信封装（GET + 自定义 headers / POST + payload，5 分钟心跳超时），支撑 Terraform plan/apply 输出、K8s 事件与日志等约 30 处实时场景；
- 集成蓝鲸 AI 助手（Vue2 版 `@blueking/ai-blueking`），设计 `aiOpenTask` Promise 串行化方案，解决「打开窗口」与「发送消息」两条并发链路导致的会话竞态与消息丢失问题，并实现 AI 输入校验（XSS 防护）；
- 参与 Monaco Editor 构建产物优化：通过语言/feature 双维度按需裁剪（仅保留 yaml），显著缩减编辑器体积；实现自定义 `layer-output` 语言高亮 Terraform 输出；
- 解决生产构建失败问题：定位 `bk-cli-service-webpack` 内置 terser + swcMinify 的 `extractComments` schema 冲突，通过 `webpack-chain .tap()` 精准修改已注册 minimizer（而非叠加新实例）修复，并沉淀完整排障注释；
- 参与性能优化：路由级懒加载 + `webpackPrefetch` 预取、路由切换自动取消未完成请求、蓝鲸 PaaS 静态资源模板替换与版本校验等部署适配；
- 参与代码质量建设：Markdown 安全渲染（marked + xss 白名单过滤）、多项目 `X-Project` 隔离、国际化（vue-i18n）。

**项目成果**（量化数据均可在代码库佐证，按实际情况保留）：

- 交付覆盖 **11 个功能模块**的前端应用，支撑 K8s 应用与 Terraform 基础设施两类资源的统一纳管；
- SSE 流式方案覆盖约 **30 处**实时输出场景，统一了长耗时任务的前端呈现方式；
- 修复阻断性的生产构建失败（terser/swc schema 冲突），保障发布流水线稳定；
- Monaco 按需裁剪有效控制编辑器产物体积，提升首屏加载表现。
