# PowerApp 应用详情子应用（new-ui）技术难点与待解决问题

## 一、技术难点

### 1. 微前端共存：Vite lib 模式 + externals 的渐进式替换方案

**难点**：旧版 Vue 2 SPA 仍在运行，新版 Vue 3 重写需要**在不中断线上服务的前提下逐页替换**；直接整站切换风险过高，且新旧页面需要共享大型依赖（monaco-editor、js-yaml 等）避免双份加载。

**方案**：一套代码、双形态产物（`vite.config.mts:108-126`）：

- **独立 SPA**：`src/main.ts:24` 挂载 `#app-detail`，独立路由（hash / history 双模式兼容宿主环境）；
- **Library 产物**：`build.lib` 入口 `src/pages/application/index.ts:19-28` 对外导出 `bkui / i18n / global / pinia / router` 模块 + `ApplicationDetail` 组件 + `createApp` + `showMessage`，宿主页按需集成；
- **依赖共享**：`rollupOptions.external` 外置 `js-yaml / monaco-editor / dayjs / diff` 并映射 globals（`vite.config.mts:110-118`），由宿主提供全局变量，lib 产物不重复打包这四个大依赖。

### 2. 原生 fetch 的拦截器与请求取消体系（无 axios）

**难点**：技术选型弃用 axios 改用原生 fetch，但原生 fetch 缺少拦截器、请求取消、统一错误处理等工程化能力，且需要支持「URL 模板变量替换」「多项目 X-Project 头注入」「部分请求不可取消」等定制需求。

**方案**：三层封装（`src/api/`）：

- `interceptors.ts:25-70`：保存 `window.fetch` 原始引用，自实现 `interceptors.request / response.use` 拦截器链；每个请求创建 `AbortController` + `uniqueId` 并注册进请求队列；
- `request-queue.ts:44-53`：`cancelRequest` 支持按 id 批量取消，过滤 `irrevocable` 配置的请求（不可取消）；路由切换时批量 abort 防止慢接口污染新页面；
- `fetch.ts:101-126`：`parseUrlAndParams` 支持 URL `$变量` 模板替换（如 `/applications/$name?project=$project`）并从 params 中剔除；`fetch.ts:154-159` 从 `window.location.pathname` / localStorage 解析项目码注入 `X-Project` 头（正则豁免特定 URL）；`interceptors.ts:12-13` `silent` 静默模式让非关键接口失败时不弹全局 toast 仅 reject。

### 3. SSE 双方案封装与断线重连管理

**难点**：原生 `EventSource` 仅支持 GET 且无法自定义请求头；后端 watch 接口需要 `X-Project` 头，部分接口还需 POST payload。SSE 长连接断开后需要自动恢复，但不能无限重连。

**方案**：

- `src/api/event-source.ts:20-37`：基于 `event-source-polyfill` 的 GET 方案（携带自定义 headers + withCredentials）；
- `src/api/event-source.ts:39-62`：基于 `sse.js` 的 POST 方案（payload JSON 序列化，有 payload 自动切 POST）；
- `src/composables/use-event-source.ts:16-19`：通用管理 composable——`MAX_RECONNECT_ATTEMPTS = 3` 次重连上限、`RECONNECT_DELAY = 5000ms` 间隔、`beforeOpen` 开启前检查、`shouldCloseOnError` 错误时是否关闭的策略化判断，统一了应用 watch / 资源树 / Pod 监控 / rollout 更新等全部 SSE 场景。

### 4. 大集群监控数据：视窗懒加载 + 分批请求 + 自适应节流

**难点**：主机视图在大集群（百级节点 / 万级 Pod）下一次性拉取全部节点监控会造成请求风暴与卡顿；监控数据 30s 级刷新进一步放大请求量。

**方案**（`use-node-pods-metrics.ts` + `stores/application.ts`）：

- **视窗懒加载**：节点卡片经 `IntersectionObserver` 命中后调用 `markNodeVisible` 注册进 `visibleNodeNames` 集合（`stores/application.ts:124-129`，一旦注册不回退避免滚动反复增删），metrics 只拉「可见且未拉取」的增量节点；
- **分批请求**：单次最多 50 个节点（与后端 `maxNodeNamesPerRequest` 对齐，`use-node-pods-metrics.ts:19`）；
- **自适应节流**：应用节点 > 100 时 60s、否则 30s（`use-node-pods-metrics.ts:101-104`）；Pod 维度同理：Pod > 1000 时 60s（`use-pod-metrics.ts:43-47`）；> 100 节点时蜂窝不带回 Pod 明细（大集群保护）；
- **数据分层覆盖**：蜂窝刷新（includePods=false）仅覆盖 usage/packing 保留 pods 详情，抽屉打开（includePods=true）才全量覆盖（`stores/application.ts:626-639`）；
- **并发去重**：`pendingMap`（Promise 复用锁）与响应式 `loadingSet`（UI 钉子）职责分离（`stores/application.ts:130-138` 有完整设计注释）。

### 5. Pinia shallowRef 响应式陷阱与 SSE 增量更新

**难点**：rollout 列表使用 `shallowRef` 存储以优化大列表性能，但 SSE 推送 patch 时若原地修改数组项属性，`v-for` 卡片、`computed.find` 等依赖完全不感知，表现为「步骤进度不实时刷新、必须强刷页面」。

**方案**：`stores/application.ts:732-757`（含完整成因注释）——SSE `update` 分支生成**全新数组**：被推送命中的项替换为新对象（新引用），未命中项保持原引用最大限度复用；snapshot 分支整体替换；同时刷新 `rolloutUpdateKey` 强制重渲染。SSE 数据合并使用 `json-merge-patch` 语义（`{ ...item, ...patch }`）。

### 6. Revision Metadata 多组件一致性缓存

**难点**：详情页「上一次同步结果」chip 与同步详情抽屉头部 chip 必须展示完全一致的 commit 信息（author / date / message），但两处组件实例不同（current=true / false），共享按索引的数组会互相覆盖；同一组 revisions 并发请求会重复发送。

**方案**：`stores/application.ts:650-730`——按 **revision SHA 为 key 的 Map** 存储 metadata；`loadRevisionMetadata` 内部：hex SHA 合法性校验（`/^[0-9a-f]{7,40}$/i`）、非法项用占位符 `'0'` 保持与 sources 位置对齐、`pendingMap` 以「排序后的 SHA 组合」为去重 key 复用并发 Promise；组件从 Map 按 SHA 读取天然一致。

### 7. AI 诊断面板：宿主注入 SDK + 流式答复结构化解析

**难点**：AI 诊断依赖宿主（旧版平台）注入的蓝鲸 AI SDK ChatHelper；流式答复是整段 markdown，需要实时拼接并解析为面板的结构化区块；解析失败不能白屏。

**方案**：`src/composables/use-ai-diagnosis.ts`——

- 通过 `inject('getAIChatHelper')` 获取宿主 SDK，`helper.session.add()` 建会话、`sendMessage()` 发 prompt，同时 `watch` message 列表实时拼接流式 content；
- markdown 按 **H1 标题切分**（`splitByH1`），章节标题归一化（容忍中英文 / 空格 / 大小写：`结论/Conclusion → conclusion`、`因果链/What Happened → whatHappened` 等）映射到 conclusion / impact / whatHappened / evidence / recommendations 五个区块；
- 解析失败时整段文本退化为 conclusion 区域展示（raw 字段兜底）；
- 输入安全：`sanitizeAIPrompt` / `isSendableAIPrompt`（`src/common/ai-input-validation.ts`，兼容旧版 `bus.$emit('aiSendMessage')` 透传）。

### 8. G6 5 网络拓扑：14 种节点类型的链路推导

**难点**：从 ArgoCD resource-tree 推导 LB → Ingress → Service → Pod 的流量链路，需覆盖异构资源：标准 K8s Ingress、BCS 自定义 Ingress、PortPool、Istio VirtualService/DestinationRule、K8s Gateway API（Gateway/HTTPRoute/TCPRoute/GRPCRoute）、Endpoint/EndpointSlice（含 ready/serving/terminating 流量状态）。

**方案**：`topo/net-topo-builder.ts`（纯 TS 数据构建器，与渲染解耦）——定义 `NetNodeKind` 14 种节点类型、`EndpointConditions` 流量三态、`podStatusSummary` 状态分布摘要与 `notReadyReasons` 汇总；`topo/` 目录 13 个 ts 配套实现 polyline（正交连线）、minimap、context-menu、tooltips、flow-line 等交互。

### 9. 时间显示全站一致性

**难点**：`dayjs.fromNow()` 内部按阈值**四舍五入**（16h30m → "17 小时前"），与 sync-header 等组件的 `Math.floor` 截断逻辑（→ "16 小时前"）冲突，同一时间戳在不同组件相差 1 个单位。

**方案**：`src/composables/use-utils.ts:31-49`（含成因注释）——弃用 `fromNow()`，`toLocalTimeForDay` 统一 `Math.floor` 截断分级（刚刚 / 分钟 / 小时 / 天 / 月 / 年），配套 `toLocalTime`（时区转换 + 空值 / 无效日期防御）、`computedTimeDiff`（双语单位）。

### 10. 样式前缀隔离：bk-prefix 三通道注入

**难点**：bkui-vue 组件库样式带 `bk-` 前缀，项目需要统一改写前缀以隔离宿主环境样式，且 less / scss / 组件库产物三条链路都要生效。

**方案**：`vite.config.mts:87-105`——less `modifyVars: { 'bk-prefix': BKUI_PREFIX }` + scss `additionalData` 变量注入 + 自研 PostCSS 插件 `plugins/css.prefix.js`（`vite.config.mts:100-104` 注册）处理编译产物，`BKUI_PREFIX` 常量统一来源于 `src/common/const`。

## 二、待解决问题

| # | 问题 | 代码依据 | 影响 |
|---|---|---|---|
| 1 | **vite-plugin-vue-devtools 被移除**：`vite-plugin-vue-inspector@5.x` 的 @babel 依赖与 pnpm 严格 ESM 隔离不兼容 | `vite.config.mts:71-73`（完整注释） | 开发期缺组件状态检视工具，只能用浏览器 Vue DevTools 扩展替代 |
| 2 | **SSR noExternal workaround**：workbox-window / vue-i18n 未支持 native ESM | `vite.config.mts:82-85`（TODO 注释） | 上游修复前需持续维护构建配置 |
| 3 | **大集群「集群全部节点」视角隐藏**：待节点视口分批 / Canvas 蜂窝 / 阈值缓存等性能优化落地后再放开 | `hosts/index.vue:40-44`（`v-if="false"` + 注释，保留逻辑便于一行恢复） | 主机视图仅可用应用视角，集群视角功能暂不可用 |
| 4 | **401 未认证处理未完成** | `src/api/fetch.ts:20-23`（`// todo 未认证`，status 401 直接 return） | 认证过期时无跳转登录引导，请求静默失败 |
| 5 | **节点运维操作待后端接口**：cordon / 驱逐等操作入口 disabled | `hosts/node-card.vue:165`、`hosts/node-detail-drawer.vue:41`（保留模板与 handleXxx，待接通后移除 disabled 即可恢复） | 节点维度运维动作前端就绪但不可用 |
| 6 | **Monaco 设置模式卡死 BUG** | `src/components/editor-diff/use-editor.ts:229`（`// todo 设置模式（卡死BUG）`） | 编辑器设置功能受限 |
| 7 | **应用外 Pod 日志 400**：「Pod xxx not found as part of application yyy」 | `hosts/index.vue:500` | 部分场景外 Pod 日志拉取失败 |
| 8 | **拓扑节点文本测量性能隐患**：ASCII 码对应像素大小计算 | `topo/resource-node.ts:279`（`todo 可能有性能问题`） | 大规模节点时拓扑渲染可能劣化 |
| 9 | **类型完善欠账**：多处 `todo 类型完善` | `table/app-table.vue:2144`、`detail.vue:639`、`@types/api.d.ts:162`、`rollout/detail.vue:599` 等 | 类型保护不完整，`vue-tsc` 检查覆盖打折 |
| 10 | **测试覆盖薄弱**：仅 2 个测试文件（基础 + flex-row 组件） | `test/basic.test.ts`、`test/component.test.ts` | 核心逻辑（stores / composables / api 封装）无测试保护 |
| 11 | **Dockerfile 端口声明不一致**：EXPOSE 5000 但 `server` 脚本监听 5001 | `Dockerfile:13` vs `package.json:11` | 容器端口映射时易误配 |
| 12 | **进行中：时间格式优化** | 当前分支 `opt-time-format` | dayjs 格式统一化改造进行中（与旧版项目同期推进） |
