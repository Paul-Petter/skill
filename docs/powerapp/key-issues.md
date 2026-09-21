# PowerApp 技术难点与待解决问题

## 一、技术难点

### 1. SSE 流式通信：双封装方案与长连接管理

**难点**：原生 `EventSource` 仅支持 GET 且无法自定义请求头，而后端接口需要 POST payload + `X-Project` 头，且大量长耗时任务（Terraform plan/apply、K8s events/logs、应用同步）依赖实时输出。

**方案**：项目维护两套 SSE 封装，约 **30 个文件**在使用：

- `src/api/event-source.js`：基于 `event-source-polyfill` 的 GET 方案（携带自定义 headers）；
- `src/common/sselib.js`：基于 `sse.js` 的 POST 方案，支持 `payload` JSON 序列化、`X-Project` 头从 `window.location.pathname` / `localStorage` 解析（`sselib.js:11`）、**5 分钟心跳超时**（`SSE_TIMEOUT`，`sselib.js:4-5`）。

### 2. Terraform Layer 多阶段生命周期建模

**难点**：`init → plan → apply → policy` 多阶段流程 + State 管理（`state-rm` / `unlock` / `refresh`）+ 批量 plan/apply + 事件流实时输出，前端需要同时处理「阶段状态机 + 流式日志 + 批量任务聚合」。

**方案**：`src/api/layer.js` 定义完整接口清单；Layer 详情页（`src/views/layer/`）实现阶段流转；**自定义 Monaco 语言 `layer-output`** 高亮 Terraform 输出（`bk.config.js:105` 注释佐证，不依赖内置 tokenizer）。

### 3. AI 助手集成：会话竞态串行化

**难点**：`watch.openResourceAI` 与 `bus.$on('aiSendMessage')` 两条并发路径都会触发「打开窗口 + 建会话」，切换项目后首次点击「AI 分析」时导致 session 竞态、消息丢失（只弹侧栏不发送）。

**方案**：

- `aiOpenTask` Promise 复用机制，把两条链路串行化（`src/components/ai-assistant/index.vue:40-46`，注释完整记录了竞态成因）；
- DOM 级等待会话创建完成：`querySelector` 查找「新增会话」按钮（class `bkai-xinzengliaotian`）并 `click()`（`index.vue:104-113`），轮询检测会话标题变化判断新会话就绪（`index.vue:115-136`，3 秒超时保底）；
- 输入安全：`src/common/ai-input-validation.js` 的 `isSendableAIPrompt` / `sanitizeAIPrompt`。

### 4. Monaco Editor 产物瘦身

**难点**：monaco-editor 0.51 全量引入体积过大。

**方案**：`MonacoWebpackPlugin` 语言/feature 双维度裁剪——只保留 `yaml` 语言（K8s YAML 场景），剔除 9 个 features（`!gotoSymbol` / `!rename` / `!colorPicker` 等），见 `bk.config.js:107-120`。

### 5. 构建工具链兼容 Hack：terser + swcMinify 的 schema 冲突

**难点**：`bk-cli-service-webpack` 内置 minimizer 为 terser-webpack-plugin + swcMinify，默认将 `extractComments: true` 透传给 swc，`@swc/core` 新版严格校验抛 `unknown field 'extractComments'`，导致**生产构建每个 chunk 报错、Build failed**。

**方案**：用 `webpack-chain` 的 `.tap()` 精准修改**已注册**的 `js` minimizer options（而非 `configureWebpack` 叠加新 minimizer——webpack-merge 会 concat 数组、无法覆盖有问题的默认项），见 `bk.config.js:35-57`（含完整推演注释）。

### 6. 路由性能：懒加载 + 请求取消

- 全部路由组件 `import()` 动态加载 + `webpackPrefetch: true` 预取高频页（`src/router/index.js:17`）；
- 路由切换时通过 `http.queue` + `cancel` 取消未完成请求，防止慢接口污染新页面数据（`src/api/` 封装）。

### 7. 蓝鲸 PaaS 部署适配

- `replace-static-url-plugin.js`：构建产物中 `{{ .BK_STATIC_URL }}` Go 模板占位符替换，适配 PaaS 静态资源注入；
- `build-version.js`：生成 `static/verify.json`，paas-server 启动时校验构建完整性；
- `env.js`：多环境 `AJAX_URL_PREFIX`，开发 HTTPS 代理（`bk.config.js:63-78`）。

### 8. Markdown 安全渲染

AI 回复 / 报告内容使用 `marked` 4.3 渲染，`github-markdown-css` 样式，`xss` 白名单过滤，防 XSS。

## 二、待解决问题

| # | 问题 | 代码依据 | 影响 |
|---|---|---|---|
| 1 | **路由 preload 机制空置**：`preload` / `fetchPageData` 导入被注释 | `src/router/index.js:8`「暂时注释：preload 函数为空，无实际功能」 | 路由级数据预取缺失，首屏数据获取时机不受控 |
| 2 | **SSE 断线重连未启用**：`onerror` 中 2 秒重连逻辑整体被注释 | `src/common/sselib.js:24-33` | 长任务（plan/apply）流式输出中断后无法自动恢复 |
| 3 | **AI 助手强依赖 DOM 操作**：`querySelector` 点击组件内部按钮、轮询标题变化检测会话 | `src/components/ai-assistant/index.vue:97-135` | `@blueking/ai-blueking` 升级改版后选择器失效，需回归验证 |
| 4 | **Vue 2 生态 EOL**：Vue 2.7（2023-12-31 停止维护）、vue-router 3.0.6、vuex 3.1.1 | `package.json` | 安全补丁停发，升级需整体迁移（组合式 API 已可用，降低了迁移成本） |
| 5 | **运行时/依赖版本陈旧**：要求 `node >= 14.17.6` / `npm 6.14.15`；依赖已废弃的 `request@2.88.2` | `package.json` engines 与 dependencies | 无法使用新版工具链，`request` 为传递依赖需排查来源 |
| 6 | **构建 Hack 依赖内部 CLI 版本**：terser/swc 兼容修改绑定 `bk-cli-service-webpack 0.0.7` 的内部实现 | `bk.config.js:35-57` | CLI 升级后需验证 minimizer 注册方式是否变化 |
| 7 | **进行中：时间格式优化** | 当前分支 `opt-time-format` | dayjs 格式统一化改造进行中 |
