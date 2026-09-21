# bk-testpilot 技术难点与待解决问题

> 基于 2026-09 代码快照整理；每条难点均标注代码 / 文档依据。

## 一、技术难点（已解决 / 已落地）

### 1. OpenAPI 驱动的前端 API 层自动生成

**难点**：前后端接口契约靠手写维护易漂移，前端需要类型安全的 API 客户端。

**方案**：以 OpenAPI 为唯一契约源。后端 `make gen` 导出 swagger JSON；前端 `scripts/gen-api-v1.cjs` 合并多份 swagger JSON（按文件名推断 tag 分域）、排除前端不需要的 `runtime` 域，生成 `src/api/modules/*.ts`（Service 调用层）与 `src/@types/*.d.ts`（类型），生成物禁止手改。

**依据**：`ui/scripts/gen-api-v1.cjs`、`ui/AGENTS.md`「API：只从生成层查找与调用」。

### 2. 统一 HTTP 客户端与全局异常拦截

**难点**：异常处理散落在各调用点会导致重复弹窗与不一致行为。

**方案**：`src/api/fetch.ts` 的 `HttpFetch` 统一封装：

- 401 → 跳转登录页；403 → 无权限提示；400 / 非 2xx → `Message` 展示错误详情（与后端 `{error, message}` 错误体对齐，兼容嵌套 `error.message`）
- URL `{var}` 占位符自动替换（`parseUrlAndParams`）
- `needRes` / `needStatus` / `originalResponse` 灵活返回形态
- `interceptorErr` 开关支持按请求关闭错误弹窗

业务侧默认不写 `try/catch`，仅在需要兜底数据 / loading 态时局部捕获。

**依据**：`src/api/fetch.ts`（含响应拦截器注册）、`ui/AGENTS.md` 第 2 节。

### 3. 数组查询参数双序列化（`queryParamsStyle`）

**难点**：GET / DELETE 数组参数的序列化需匹配后端 OpenAPI `style: form` 的 `explode` 定义，JS 默认逗号拼接与后端期望可能不一致。

**方案**：支持 `'comma'`（默认，`?action=create,update`）与 `'exploded'`（`?action=create&action=update`）两种策略，按接口声明传 `{ queryParamsStyle: 'exploded' }`。

**依据**：`src/api/fetch.ts`（`objectToExplodedQueryParams` 分支）、`ui/README.md`「数组参数序列化」专节。

### 4. 运行时环境变量注入（一份构建产物适配多环境）

**难点**：构建产物中的环境相关变量（登录地址、API 网关等）若构建期固化，则一个环境需要一次构建。

**方案**：`index.html` 使用 `__BTP_RT_BK_*__` 占位符；自定义 Vite 插件 `html-runtime-env-replace` 仅在 `serve`（本地开发）时替换为 `.env.development` 值；生产构建保留占位符，由容器启动脚本 `docker-entrypoint.sh` 在运行时注入，实现同一镜像多环境部署。

**依据**：`vite.config.mts`（`html-runtime-env-replace` 插件及注释）、`docker/` 目录、`ui/README.md`「环境变量」一节。

### 5. 多租户用户选择器接入（CORS 与占位符替换）

**难点**：`@blueking/bk-user-selector` 需直连用户管理服务；多租户 / 非多租户环境 URL 规则不同，开发环境存在跨域问题。

**方案**：开发环境经 Vite 反向代理（`/bk-user-selector` → `BK_USER_URL`，并重写 `sec-fetch-*` 请求头与 cookie domain）规避 CORS；生产环境多租户时 `BK_USER_URL` 含 `{api_name}` 占位符，前端替换为 `bk-user-web` / `prod`，非多租户直接使用。

**依据**：`vite.config.mts` 代理配置、`src/components/user-selector.vue`、`ui/README.md`「人员选择器环境变量」专节。

### 6. Design Tokens 主题体系

**难点**：颜色硬编码散落各页面难以统一维护。

**方案**：三级链路——`src/styles/tokens/colors.css` 定义 `--bk-tp-{group}-{step}`（对齐 Figma 蓝鲸色板）→ `variables.css` 汇总 → `uno.config.ts` 映射为 UnoCSS 原子类（`bg-brand-2`、`text-text-1`、`border-neutral-6` 等）与 shortcut（`bk-tp-card` 等）；页面禁止硬编码 hex / rgb。

**依据**：`src/styles/tokens/`、`uno.config.ts`、`ui/AGENTS.md` 第 6 节。

### 7. 表格单元格截断问题（`CellLink` 组件）

**难点**：bkui-vue `<Button text>` 的 `display: inline-flex` 不受 `text-overflow: ellipsis` 控制，列宽不足时截断异常。

**方案**：基于 `<span>` 实现 `CellLink` 组件替代表格内的文本按钮，`text-overflow` 天然生效；仅限表格列内场景，页面级操作按钮仍用 `<Button>`。

**依据**：`src/components/cell-link.vue`、`ui/README.md`「表格内可点击文本」专节。

### 8. 布局与导航元信息系统

**难点**：空间级二级侧栏、内容区标题栏、导航高亮需要在路由层统一声明。

**方案**：`vite-plugin-vue-layouts`（默认 `space-side` 布局，另有 `main` / `space` / `default`）+ 路由 `meta` 约定（`layout` / `menuId` / `sideMenuId` / `contentHeader` / `contentClass`）；无二级菜单配置时侧栏自动隐藏。

**依据**：`src/modules/router.ts` 头部 meta 约定注释、`src/layouts/`（4 个布局文件）。

### 9. 多空间切换的状态隔离与视图刷新

**难点**：URL 携带 `:space` 参数，直接切换空间时需校验空间合法性并强制刷新空间级页面状态。

**方案**：路由 `beforeEach` 中 `spaceStore.resolveSpace()` 校验空间状态（非 `active` 跳 404）；`afterEach` 中跨空间切换调用 `bumpRouteViewKey()` 强制重建视图；另实现 `smartGoBack`（无历史栈时回退到父级路由而非浏览器 back）。

**依据**：`src/modules/router.ts`（路由守卫与 `smartGoBack` 实现）、`src/stores/space.ts`。

### 10. 后端执行链路：单一写权威与快照冻结

**难点**：分布式执行（MQ 下发 + 沙箱执行）下的状态一致性：重复上报、乱序、超时、重试并发。

**方案**（后端设计，前端通过 API 消费其状态）：

- Plan 是 Task / Session 状态的唯一写者，Worker 只管理执行环境，不写业务状态
- Runtime 上报校验一次性 Token 与 Task / Run 绑定，使用幂等键 + 条件更新抵御重复与乱序
- Run 创建时冻结用例、数据集、被测环境与执行环境快照；报告只读快照，保证历史报告稳定
- 心跳超时由 Plan 的 `SweepHeartbeatTimeouts` 探测并置 `timeout`，随后下发 abort

**依据**：`docs/development/services/test-plan-execution.md` 及其引用的拆分契约文档。

### 11. 国际化文案一致性

**难点**：placeholder 类文案（「请搜索 xxx」）若逐条入词典会产生大量重复词条。

**方案**：YAML 词典仅存 labels，`use-search-placeholder` 等 composable 统一拼接前缀；`@intlify/unplugin-vue-i18n` 构建期编译 `zh-CN` / `en-US` 语言包。

**依据**：`src/composables/use-search-placeholder.ts`、`vite.config.mts` VueI18n 配置、`src/locales/`。

## 二、待解决问题与风险

### 1. 测试数据模块处于「已开发但隐藏」状态

`src/pages/space/data/` 已有 16 个文件（14 `.vue` + 2 `.ts`），但路由被注释隐藏，注释为「测试数据暂时不开发，暂时隐藏」（`src/modules/router.ts`）。后续上线需恢复路由并回归验证。

### 2. 分布式执行链路的测试维护负担

执行链路涉及幂等、乱序、超时、取消、重试等边界场景，后端文档将「是否补充幂等、乱序、超时、取消和重试测试」列入变更检查清单（`docs/development/services/test-plan-execution.md` §5），每次协议变更需持续维护，存在回归风险。

### 3. 生成物同步依赖流程纪律

前端 API 层依赖「后端 `make gen` → 前端 `pnpm gen:api:v1`」的人工按序执行；虽有「禁止手改生成文件」约束，但若跳过同步步骤会导致前后端契约漂移。

### 4. 人员选择器环境适配复杂

`BK_USER_URL` 需按「开发 / 生产多租户 / 生产非多租户」三种环境维护不同配置（占位符替换规则各异，见 `ui/README.md` 专节），部署配置出错会直接影响人员选择功能可用性。

### 5. 前端无自动化测试体系

`ui/package.json` 无测试框架依赖与 test 脚本，质量保障依赖 vue-tsc 类型检查 + ESLint + lint-staged（含 staged 类型检查）；后端有 `make test` / `make test-integration`，前端暂无对应的单测 / E2E 保障。

> 说明：代码库中 TODO / FIXME 标记极少（全库检索仅 7 处命中且均为文档性注释），待解决问题主要来自「已隐藏功能」与「流程性风险」，而非未完成代码。
