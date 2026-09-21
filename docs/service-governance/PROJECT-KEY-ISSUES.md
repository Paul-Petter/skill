# bkms-ui 技术难点与待解决问题

> 依据当前代码仓库整理，每条标注来源（文件路径 / 文档章节）便于溯源；表述基于代码与文档现状。

## 一、技术难点

### 1. 基于 @antv/g6 v5 + g6-extension-vue 的 K8s 资源拓扑图

**难点**：在 Canvas 图引擎中渲染 Vue 组件节点，并实现数据高频刷新下的高性能交互式拓扑。

**当前实现**：

- Vue 组件作为自定义节点：`topology-node.vue` 经 `g6-extension-vue` 注册为 `resource-node`（`register(ExtensionCategory.NODE, ...)`，用标志位防重复注册）
- 两套自定义边（`custom-edge.ts`）：`PrimaryEdge` 继承 Polyline，`getKeyPath()` 手工计算含圆弧转角（`A` 命令）的折线路径；`AuxiliaryEdge` 继承 CubicHorizontal，虚线贝塞尔表达非核心依赖
- 自定义 Behavior：`ShowAuxiliaryEdgesOnHover` —— 辅助边不参与布局，hover / click 时按需 `addEdgeData` / `removeEdgeData` 动态增删
- 性能手段：`shallowRef` 持有 Graph 实例（避免 Vue 深度代理 G6 内部对象）、`useWorker: true` 布局计算、diff 增量更新（`diffArrayFast`）替代全量 `render()` 消除布局抖动、4 秒轮询 + 全局动画关闭
- 显示质量：节点内 CSS `zoom: 4` + `transform: scale(0.25)` 四倍超采样，解决 HTML 节点放大后文字模糊
- 交互细节：自定义 minimap 形状（`@antv/g` Group + Rect）；弃用 G6 内置 tooltip 插件，改用 Vue 组件 + `getElementRenderBounds` + `getViewportByCanvas` 坐标换算定位到边中心

**沉淀**：模块内 `SUMMARY.md` 记录 10 条避坑指南（状态覆盖语义、坐标转换方向、空数组过滤语义、辅助边被 diff 误删等）。

**依据**：`src/pages/application/components/topo/`（index.vue、resource-topology.vue、custom-edge.ts、topology-node.vue、SUMMARY.md 等）

### 2. Swagger 驱动的 API / 类型自动生成链路

**难点**：前后端接口契约同步，避免手写接口与类型定义的维护成本和漂移。

**当前实现**：`scripts/gen-api-v1.js` 读取仓库内 `bkms-server/docs/apis/swagger.json`，经 `scripts/openapi-v1/generator.cjs` 生成 `src/api/modules/v1/`（42 个 API 模块文件）与 `src/@types/v1/`（76 个类型文件）；后端接口变更后重新执行 `pnpm gen:api:v1`，生成物禁止手工修改（README、docs/API_GUIDE.md 均有约定）。

**依据**：`scripts/gen-api-v1.js`、`docs/API_GUIDE.md` §3.1

### 3. 自研 Fetch 请求基础设施

**难点**：统一多后端（bkms-server / BCS / 制品库 / 用户选择器）请求的响应处理、错误提示与链路追踪。

**当前实现**：

- `ConsoleFetch` 封装类提供类型安全的 get/post/put/delete/patch，URL 中 `{var}` 路径参数自动替换
- 统一响应拦截：401 跳转登录页（`BK_LOGIN_URL?c_url=当前URL`）、403 无权限提示、400 错误详情附带 traceId
- 请求队列（`request-queue.ts`）：基于 `AbortController`，路由切换自动取消未完成请求，`irrevocable` 标记例外
- 链路追踪：请求侧 `X-Bkapi-Request-Id` 与响应侧 `X-Trace-Id` 分离处理
- 配置化能力：`needRes` / `originalResponse` / `multipart` / `responseType: 'blob'` / `validateCode` 等扩展字段

**依据**：`src/api/fetch.ts`、`src/api/interceptors.ts`、`src/api/request-queue.ts`、`src/api/trace-id.ts`、`docs/API_GUIDE.md`

### 4. 「一次构建、多环境部署」的运行时配置注入

**难点**：静态 SPA 产物无法在部署期修改配置，又要避免为每个环境单独构建镜像。

**当前实现**：`.env.production` 中以 `__BKMS_RT_BK_XXX__` 占位符形式将变量打包进产物；容器启动时 `docker/docker-entrypoint.sh` 扫描产物中的占位符并用实际环境变量替换（未提供则清空），随后启动 Nginx。支持挂载 `.env` 文件、单个 `-e`、`--env-file` 三种注入方式，`-e` 优先级最高；与构建时变量（`BKMS_APP_VERSION` 等）分层管理。

**依据**：`DEPLOY.md`、`docs/ENV_VARIABLES.md`、`docker/docker-entrypoint.sh`

### 5. URL query 双向同步机制（use-url-query-sync）

**难点**：页面状态（Tab 锚点、筛选条件）与 URL 双向同步，且要处理多值参数、非法值收敛、跨页参数污染等边界。

**当前实现**（`src/composables/use-url-query-sync.ts`，约 220 行）：

- 泛型配置驱动：单值 / 数组双模式由 `default` 类型自动推断；`allowed` 合法值校验；`override` 自定义覆盖（返回空串表示该字段不参与同步）
- 挂载时一次性 reconcile：逐字段判定 skip / remove / write，差异合并为一次 `router.replace`
- 卸载时刻意不清理 query（避免切应用时卸载 replace 打断 push），跨菜单参数由 `detail.vue` 传 `query: undefined` 清理
- 与路由守卫配合：观测页专用参数（`apmQuery` / `env`）只允许存在于对应观测页，`src/modules/router.ts` 全局前置守卫按目标页逐个剔除

**依据**：`src/composables/use-url-query-sync.ts`、`src/modules/router.ts`、`README.zh-CN.md`

### 6. 菜单驱动的动态路由与导航体系

**难点**：同一详情壳下按应用类型（tRPC / TAF / Helm / Agones）渲染不同菜单树与业务组件。

**当前实现**：

- `src/config/navigation/app.ts` 维护 `TRPC_NAVIGATION`（taf / trpc 共用）与 `HELM_NAVIGATION`（helm / agones 共用）两套配置，`NavigationItem` 直接挂组件引用
- `CustomRouterComponent` 根据路由参数与导航配置动态加载业务组件，env / basic / plugin / platform 菜单页同样复用该机制
- 覆写 `router.back()`：无浏览历史时自动从 `route.matched` 推导上级页面（`resolveParent` + `smartGoBack`）
- 全局守卫做空间有效性校验：空间不在可访问列表 → 403；空间未就绪 → 404；空间切换时重建环境上下文

**依据**：`src/modules/router.ts`、`src/config/navigation/`、`src/components/custom-router-component.vue`、`docs/ARCHITECTURE.md` §5.1

### 7. 第三方监控组件的 ECharts 注册时序问题

**难点**：`@blueking/monitor-vue3-components` 内部通过 `echarts/core` 的 `init()` 初始化图表但未注册任何 Renderer / Component，直接使用会抛 "Renderer 'undefined' is not imported"。

**当前实现**：`src/main.ts` 在应用启动时全局 `echarts.use()` 注册 CanvasRenderer、LineChart 及所需组件（包与项目共享同一 echarts 实例）；同时为依赖 `window.timezone` 的 DateRange 组件兜底本地 IANA 时区，避免图表刷新解析出错卡死。

**依据**：`src/main.ts`（含问题成因注释）

### 8. Playwright + BDD 的 E2E 测试体系

**难点**：让业务可读的 Gherkin 用例与稳定的 UI 自动化并存，并支持 AI 辅助生成与 CI 容器化运行。

**当前实现**：四层架构 —— `.feature`（语义层）→ `*.steps.ts`（桥接层，禁止出现 selector）→ `*.action.ts`（业务流程组合）→ `*.page.ts`（Page Object 原子操作）+ 表单 Schema 引擎；配套 `bkms-bdd-gen` skill 由 AI 生成用例、`run.sh` 一键执行（bddgen → 测试 → HTML 报告 → 可选上传 BKRepo）、node:22 + Chromium 容器镜像、按 `@smoke` / `@deploy-flow` / `@config-flow` / `@readonly` 分 profile 运行。

**依据**：`e2e/README.md`、`e2e/` 目录（17 个 `.feature`）

## 二、待解决问题

| # | 问题 | 现状与依据 |
| --- | --- | --- |
| 1 | 应用详情「概览」页未实现 | `src/pages/application/detail/overview.vue` 为空模板（`<!-- overview todo -->`），且导航配置中被注释隐藏（`src/config/navigation/app.ts` 注释块标注「HIDE，等待后续页面开发后再展示」） |
| 2 | 监控告警日志页未实现 | `src/pages/application/detail/alert-log.vue` 为空模板（`<!-- alert todo -->`） |
| 3 | 拓扑「资源列表」模式未实现 | `src/pages/application/components/topo/resource-list.vue` 仅占位文本（`TODO: 资源列表`），文件职责表中已规划为「资源列表模式」 |
| 4 | 组件批量删除逻辑未实现 | `src/pages/plugin/component-config/component-list.vue:863` 存在被注释的 TODO 代码块（`// TODO: 实现批量删除逻辑`） |
| 5 | Helm 源配置 BCSRepo 分支待补充 | `src/pages/application/detail/base-info/helm/helm-source-config.vue:114`（`TODO: BCSRepo 处理逻辑待补充`） |
| 6 | GitSelector 组件仅支持 TGit | `src/pages/application/template/helm-chart/helm-chart-build-form.vue:226、257` 两处硬编码 `type: 'TGit'`（注释 `todo 目前GitSelector组件只支持TGit`），其他 Git 托管类型暂不可选 |
| 7 | 请求队列自定义 id 入队 / 移除不一致 | `docs/API_GUIDE.md` §7 明确：当前自定义 id 不能作为请求合并或缓存键，修复前不要依赖其做去重或生命周期管理 |
| 8 | 应用挂载强依赖用户信息接口 | `docs/ARCHITECTURE.md` §3 / `src/main.ts`：应用创建发生在 `getUser` 成功之后，用户信息接口失败会阻断整个应用挂载 |
| 9 | 运行时变量注入存在三项限制 | `docs/ENV_VARIABLES.md` §3.2：`BK_API_BASE_URL` 不会进入生产静态产物；`BK_API_V1_PREFIX` 无占位符且 entrypoint 正则不匹配含数字变量名；`BK_BSCP_URL` 生产配置固定为空、不能运行时覆盖 |
| 10 | 生产镜像未内置 CSP / HSTS | `DEPLOY.md` §6：CSP 需按业务与外链单独设计；HSTS 建议在 TLS 终止层（网关 / CDN）下发 |

> 说明：第 1–6 条为代码中的显式 TODO；第 7–10 条为文档中标注的已知限制与设计取舍。
