# 简历项目经历：蓝鲸服务治理平台 bkms-ui（Web 前端）

> 使用说明：**时间、角色为占位符，请自行替换**；职责条目均基于仓库可验证的实现撰写，请按实际参与情况增删条目、调整动词（本文使用「参与 / 负责 / 实现」，避免夸大表述）。

## 项目经历

**项目名称**：蓝鲸服务治理平台 bkms-ui（Web 前端，蓝鲸智云开源项目，MIT License）

**时间**：XXXX.XX – XXXX.XX（请自行替换）

**角色**：前端开发工程师（请自行替换）

**技术栈**：Vue 3 · TypeScript · Vite · Pinia · Vue Router · Vue I18n · UnoCSS · bkui-vue · @antv/g6 · Monaco Editor · ECharts · Playwright BDD · Docker / Nginx

### 项目描述

蓝鲸智云服务治理平台（BlueKing Service Governance）的 Web 前端，提供微服务从构建、制品、部署到可观测的全生命周期管理界面，支持 tRPC / TAF、Helm、Agones 四类应用类型。覆盖应用管理、部署环境、组件市场、空间设置、平台管理等 8 个业务模块，`src/` 下含 339 个 Vue 组件与 247 个 TypeScript 文件、42 个自动生成的 API 模块；通过 Docker 多阶段构建实现「一次构建、多环境部署」。

### 职责与成果

1. 参与 API 层建设：实现基于后端 swagger.json 的 API 模块与 TypeScript 类型自动生成链路（42 个 API 模块 + 76 个类型定义文件），保障前后端接口契约同步；封装自研 Fetch 请求基础设施，统一 401 / 403 / 400 响应拦截、traceId 链路追踪、路由切换自动取消请求与 multipart / blob 下载等能力。
2. 负责 K8s 资源拓扑图：基于 @antv/g6 v5 + g6-extension-vue 实现 Vue 自定义节点、圆弧折线 / 虚线贝塞尔两套自定义边、hover 按需加载辅助边；通过 4 倍超采样解决 Canvas HTML 节点文字模糊，diff 增量更新 + Web Worker 布局消除高频轮询下的渲染抖动，并沉淀 10 条避坑实践文档。
3. 参与部署方案落地：实现「一次构建、多环境部署」——构建产物保留 `__BKMS_RT_BK_XXX__` 占位符、容器启动时注入运行时配置，生产镜像采用 `node:23-alpine` 构建 + `nginx:1.30-alpine` 运行，配套健康检查与缓存 / 安全头策略。
4. 实现菜单驱动的动态路由体系：tRPC / TAF 与 Helm / Agones 两套导航配置 + CustomRouterComponent 动态渲染；覆写 `router.back()` 实现无历史时的智能返回；全局路由守卫完成空间权限校验（不可访问 → 403、未就绪 → 404）与观测页专用参数的跨页清理。
5. 参与搭建 E2E 测试体系：Playwright + playwright-bdd 四层架构（feature / steps / actions / Page Object + 表单 Schema 引擎），17 个 Gherkin 用例覆盖冒烟 / 部署 / 配置 / 制品流程，支持按标签分 profile 运行、容器化 CI 执行与 HTML 报告归档。
6. 参与工程质量建设：ESLint / Stylelint / Prettier / Biome + husky + lint-staged 提交卡点，vue-tsc 全量与增量类型检查，zh-CN / en-US 双语言国际化。

### 量化数据（均有仓库依据）

| 指标 | 数值 |
| --- | --- |
| 业务模块 | 8 个 |
| 跨页面复用组件 / composables / Pinia Store | 91 / 45 / 11 |
| 自动生成 API 模块 / 类型定义文件 | 42 / 76 |
| BDD E2E 用例 / 单元测试文件 | 17 / 9 |

> 提示：量化数据为项目整体规模，写入简历前请结合个人实际贡献范围调整表述。
