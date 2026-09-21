# 简历项目经历：PowerApp 应用详情子应用（Vue 3 重构）

> 说明：时间、角色、团队规模等占位项以 `【】` 标注，请自行替换；所有量化数据均可在代码库中佐证，未夸大。

---

**项目名称**：PowerApp 应用详情子应用 —— K8s 应用管理平台前端 Vue 3 重构（new-ui）

**项目时间**：【20XX.XX – 至今，请自行替换】

**技术栈**：Vue 3.4 / TypeScript 5.4 / Vite 6（Library 模式微前端）/ Pinia / Vue Router / vue-i18n / bkui-vue（蓝鲸 Vue3 组件库）/ UnoCSS / @antv/g6 5 / Monaco Editor / xterm.js / SSE（sse.js + event-source-polyfill）/ 原生 fetch / Vitest / pnpm

**项目描述**：

PowerApp 是蓝鲸体系下的 Kubernetes 应用管理平台（GitOps 风格）。本项目是其核心「应用详情」场景的 **Vue 3 + TypeScript 全面重写**：覆盖应用同步 / Diff / 事件 / 部署历史 / 发布管理（Rollout）、资源树与网络拓扑可视化、主机视图（节点卡片 + 蜂窝布局 + 节点 / Pod 监控 + Web 终端）与 AI 诊断等能力，并集成蓝鲸 AI 助手。采用 **Vite Library 模式双形态构建**——既可独立部署为 SPA，也可产出 lib 包嵌入存量 Vue 2 宿主页实现**渐进式替换**，与宿主共享 monaco-editor / js-yaml 等大型依赖。前端约 **218 个源文件（128 个 Vue 组件）**、**18 个组合式函数**、**78 个 API 接口定义**，支持多项目隔离（X-Project）与中英国际化。

**主要职责**（按实际情况增删）：

- 负责应用详情子应用的前端架构设计与核心模块开发，基于 Vite lib 模式 + externals 依赖外置实现新旧版本共存方案，在不中断线上服务的前提下完成 Vue 2 → Vue 3 渐进式替换；
- 基于原生 fetch 重新设计请求层（替代 axios）：自实现拦截器链 + AbortController 请求队列（支持路由切换批量取消、irrevocable 不可取消标记）、URL `$变量` 模板替换、多项目 `X-Project` 头注入、silent 静默错误模式；
- 实现 SSE 双方案流式通信（event-source-polyfill GET / sse.js POST）与统一重连管理（3 次上限 + 5s 间隔 + 策略化关闭），支撑应用 watch、资源树、Pod 监控、发布单进度等实时更新场景；
- 设计大集群监控数据方案：IntersectionObserver 视窗懒加载注册 + 单批 50 节点分批请求 + 节点数 / Pod 数自适应节流（>100 节点 / >1000 Pod 时 60s）+ pendingMap 并发去重与响应式 loading 分层，解决百级节点 / 万级 Pod 下的请求风暴；
- 解决 Pinia shallowRef 响应式陷阱：SSE 增量更新采用「新数组 + 新对象引用」整体替换策略（json-merge-patch 语义），修复发布单步骤进度不实时刷新的问题并沉淀成因注释；
- 实现嵌入式 AI 诊断面板：通过 provide/inject 获取宿主注入的蓝鲸 AI SDK，流式答复按 H1 标题切分解析为结论 / 影响面 / 因果链 / 证据 / 建议动作五区块（解析失败降级兜底），配套 AI 输入安全校验；
- 参与网络拓扑能力开发：基于 @antv/g6 5 从 ArgoCD resource-tree 推导 LB → Ingress → Service → Pod 流量链路，覆盖标准 Ingress、BCS Ingress、PortPool、Istio、Gateway API、EndpointSlice 等 14 种节点类型；
- 参与工程化建设：bk-prefix 样式前缀三通道注入（less / scss / 自研 PostCSS 插件）、monaco 五类 worker 内联预构建、组件自动注册（unplugin-vue-components）、Docker 多阶段构建（pnpm store 缓存）、统一相对时间 Math.floor 截断策略保证全站展示一致。

**项目成果**（量化数据均可在代码库佐证，按实际情况保留）：

- 交付覆盖 **7 个 Tab 模块**（概要 / 详情 / 历史 / 事件 / 主机 / 网络 / 发布）的应用详情子应用，约 **218 个源文件、128 个 Vue 组件**；
- lib 模式 + externals 方案落地，与宿主共享 4 个大型依赖（monaco-editor / js-yaml / dayjs / diff），实现零中断的渐进式替换路径；
- 请求层全量替代 axios 并补齐工程化能力（拦截器 / 取消队列 / 静默模式），接口模块沉淀 **78 个**类型化定义；
- 大集群监控方案将节点监控请求收敛为「视窗内增量 + 50/批」，配合自适应节流显著降低请求量；
- 修复 shallowRef 增量更新失效问题，发布单进度实现秒级 SSE 实时刷新；
- 统一全站时间展示口径（Math.floor 截断），消除同一时间戳在不同组件相差 1 单位的不一致。
