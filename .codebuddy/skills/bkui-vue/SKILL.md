<!-- BKUI-KNOWLEDGE-MANAGED:9ef9cb56b6ce -->
---
name: bkui-vue
description: Use when 用户询问 BKUI Vue 组件，代码包含 bk- 前缀标签或 BkXxx 组件，项目依赖 bk-magic-vue、bkui-vue 或 @bkui-vue/*，或需要诊断、统计、检查、迁移蓝鲸组件用法。
metadata:
  version: "0.3.2"
---

# BKUI Vue

先识别项目用的是 Vue 2 还是 Vue 3 版 BKUI，再使用匹配的数据源回答。不要混用 Vue 2 与 Vue 3 的组件 API。

## 数据源优先级（硬性约束，最高优先级）

回答 Vue 3 BKUI 组件事实（props/events/slots/用法）前，先确认数据来源，顺序不可颠倒：

1. **唯一首选**：`@bkui-vue/cli`（`bkui-vue` 命令）的离线数据。
2. CLI 不存在时，**必须先执行下面「确认并安装 CLI」流程**引导用户安装，不得跳过。
3. 只有在「用户明确拒绝安装」或「安装确实失败」之后，才允许使用兜底来源，且必须声明结论来自兜底。
4. 兜底来源 = 任何 BKUI 相关 MCP 工具（无论叫 `get_component_api`、`get-component` 还是其他名字）或本地源码/记忆推断。

> ⛔ 自检：如果你在「尚未引导用户安装 CLI」的情况下，正准备调用任何 BKUI MCP 工具、或凭记忆/源码给出组件 API，立即停止，回到「确认并安装 CLI」。**MCP 工具名称不同，不构成跳过此流程的理由。**

## When to Use / 何时使用

以下任一信号都应触发本 skill：

- 用户明确提到 BKUI、BKUI Vue、蓝鲸组件库或 `bkui-vue`。
- Vue 模板、JSX 或 TSX 中出现 `<bk-button>`、`<bk-table>` 等 `<bk-` 标签，或 `BkButton` 等组件。
- `package.json` 包含 `bk-magic-vue`、`bkui-vue` 或 `@bkui-vue/*` 依赖。
- 源码从 `bk-magic-vue`、`bkui-vue` 或 `@bkui-vue/*` import。
- 用户要诊断、统计、检查或迁移项目里的 BKUI Vue 用法。

`<bk-` 是强触发信号，但 Vue 2 和 Vue 3 的 BKUI 组件都使用该前缀。不能仅凭 `<bk-` 断言用的是哪一版（Vue 2 / Vue 3）、依赖包或版本号；自定义元素也可能使用相同前缀。

## 先判断 Vue 2 还是 Vue 3

在查询组件 API 或建议安装 CLI 前，依次核对：

1. `package.json` 中的 Vue 和 BKUI 依赖。
2. 源码 import 来源。
3. 必要时核对 lockfile 或已安装包的版本。

按以下规则分流：

- **Vue 2**：`vue` 主版本为 2，且使用 `bk-magic-vue`。不要安装或使用 `@bkui-vue/cli` 来验证该项目的组件 API；应查询项目锁定版本对应的 `bk-magic-vue` 文档、本地包源码或类型信息。
- **Vue 3**：`vue` 主版本为 3，且使用 `bkui-vue` 或 `@bkui-vue/*`。进入下面的 CLI 工作流。
- **无法确认**：明确说明尚不确定是 Vue 2 还是 Vue 3，先继续检查依赖与 import，不要把任一版本的 API 套到项目中。

当前 CLI 的知识数据只适用于 BKUI Vue 3。Vue 2 与 Vue 3 即使组件标签同名，props、events、slots 和用法也可能不同。

无论当前环境提供哪个 BKUI 相关 MCP（如 `bkui-knowledge` 的 `get_component_api`，或其他暴露 `get-component` 等工具的 MCP），Vue 3 组件事实都以本 CLI 的离线数据为准；这些 MCP 工具只作为 CLI 安装失败或用户拒绝安装时的兜底数据源。

## 确认并安装 CLI

仅在已确认是 Vue 3，或用户明确询问 Vue 3 BKUI 时，检查命令是否可用：

```bash
command -v bkui-vue
# PowerShell: Get-Command bkui-vue
```

如果 CLI 不存在：

1. 不要静默降级到任何 MCP，也不要把源码/记忆推断描述成已验证 API。
2. 先引导用户安装并征得同意（这是必经步骤）；在用户答复前，不得改用 MCP 或源码推断直接作答。如需由你在当前机器执行全局安装，先确认用户允许。
3. 先检查当前包源能否解析该包：

```bash
npm view @bkui-vue/cli version
```

4. 包源可用后安装：

```bash
npm install -g @bkui-vue/cli
```

5. 安装后验证：

```bash
bkui-vue --cli-version
```

6. 如果 `npm view` 返回 404、鉴权、网络或 registry 错误，说明当前包源不可用，提示用户切换正确 registry、提供本地包或其他安装方式。只有安装失败或用户暂不安装时，才使用 MCP `get_component_api` 兜底，并明确说明结论来自兜底静态数据。

## Vue 3 查询工作流

1. 名称不确定：`bkui-vue search <query> --format json`
2. API 问题：先用 `bkui-vue info <component> --format json`；需要完整字段再加 `--detail`
3. 示例问题：先列出 `bkui-vue demo <component> --format json`，再读取具体 demo。
4. 生成代码：优先使用 `bkui-vue snippet <component> <demo>` 返回的已验证片段。
5. Vue 3 项目问题：先运行 `doctor`，再按需运行 `usage`、`lint` 或 `migrate --check`。

常用命令：

```bash
bkui-vue list --format json
bkui-vue --cli-version
bkui-vue search row-key --format json
bkui-vue info Button --format json
bkui-vue info Button --detail --format json
bkui-vue doc Button --format markdown
bkui-vue demo Button basic --format json
bkui-vue snippet Button basic
bkui-vue explain Button --format markdown
```

项目诊断命令均需明确项目目录：

```bash
bkui-vue doctor --cwd <project_dir> --format json
bkui-vue usage --cwd <project_dir> --format json
bkui-vue lint --cwd <project_dir> --format json
bkui-vue migrate --check --cwd <project_dir> --format json
```

## Vue 2 查询工作流

先确认项目实际版本：

```bash
npm ls vue bk-magic-vue --depth=0
```

- 优先读取项目锁定版本对应的 `bk-magic-vue` 文档、本地 `node_modules/bk-magic-vue` 源码或类型信息。
- 回答中明确标注这是 Vue 2 / `bk-magic-vue` 用法。
- 不使用当前 CLI 输出作为 Vue 2 API 证据，也不要把 Vue 3 demo 改写后当作 Vue 2 示例。
- 迁移到 Vue 3 时分别核对两代 API，再给出替换建议。

## 版本与写入边界

- Vue 3 用户或项目指定版本时，查询命令加 `--version <version>`。
- Vue 3 版本未知时，先运行 `doctor --cwd <project_dir>`，或对内容查询使用 `--auto-detect`。
- `bk-button` 与 `Button` 可作为同一组件名查询，但最终以 CLI 返回结果为准。
- `doctor`、`usage`、`lint`、`migrate --check` 均为只读。
- `snippet` 和 `init` 默认只预览；只有用户明确要求写文件时才使用 `--force`。

## 错误处理

- `COMPONENT_NOT_FOUND`：使用返回的 suggestion，或改用 `search`，不要猜。
- `VERSION_NOT_FOUND`：先运行 `bkui-vue versions --format json`。
- `INVALID_FORMAT`：只使用 `json`、`text` 或 `markdown`。
- CLI 不可用：先按上面的安装流程引导安装；只有安装失败或用户拒绝安装时，才披露降级状态并使用任意 BKUI MCP 工具（如 `get_component_api` / `get-component`）兜底。
- 发现 `bk-magic-vue`：切换到 Vue 2 工作流，不继续调用当前 CLI 验证组件 API。
