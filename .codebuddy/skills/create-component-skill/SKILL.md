---
name: create-component-skill
description: >
  bk-base-v4 组件 Skill 生成器。用于分析 bk-base-v4 中的组件并生成/更新结构化 Skill 文档（SKILL.md + examples.md）。
  触发：为组件创建 Skill、生成组件 Skill、更新组件 Skill、写 Skill 文档、组件改了更新 Skill、补充组件使用文档。
---

# create-component-skill

本 Skill 是一份"Skill 生成器"指引，告诉 AI 在遇到"为 bk-base-v4 组件生成或更新 Skill 文档"需求时应如何执行。

支持两种工作流：

- **工作流 A：新建 Skill** — 组件尚无 Skill 文档时，从零生成
- **工作流 B：更新已有 Skill** — 组件已改动，同步更新已有 Skill

## 命名与目录约定

| 项目 | 规范 |
|------|------|
| Skill 目录名 | `use-{component-kebab-name}`（如 `use-editable-tags`） |
| 存放位置 | `.cursor/skills/use-{name}/` |
| 必须文件 | `SKILL.md` |
| 推荐文件 | `examples.md`（复杂组件必须有） |
| frontmatter name | 与目录名一致 |
| frontmatter description | 非空，表达功能 + 触发场景关键词 |

遵循 `cursor-governance` 规则。

## 复杂度判断

生成前先判断组件复杂度，选择合适风格：

| 复杂度 | 判断标准 | SKILL.md 风格 | examples.md |
|--------|----------|---------------|-------------|
| 简单 | Props ≤ 4 且无 Events/Slots/Expose | 简洁编号列表（参考 `use-editable-tags`） | 可选 |
| 复杂 | Props > 4 或有 Events/Slots/Expose | 分步骤详解 + 禁止项（参考 `use-bkbase-list-table`） | 必须有 |

---

## 工作流 A：新建 Skill

### 阶段 A1：分析组件源码

1. 读取组件 `.vue` 文件（完整读取，不要截断）
2. 提取以下 API：
   - **Props**：名称、类型、默认值、是否必填、用途（从代码注释或命名推断）
   - **Events**：名称、参数类型、触发时机
   - **Slots**：名称、用途
   - **Expose**：方法名、参数、返回值
3. 识别**内部依赖组件**（子组件 import）
4. 注意**国际化 key**：若默认 placeholder 等使用了 `DATAFLOW:` 等前缀的 i18n key，标注出来
5. 如组件通过 `@/components/index.ts` 统一导出，确认导出名称

### 阶段 A2：搜索使用场景

使用 `code-explorer` subagent 或 `search_content` 工具：

1. 搜索 `import 组件名 from` 或 `import { 组件名 } from '@/components'` 找到所有引用
2. 对每个引用文件，读取关键代码片段（5~20 行上下文）
3. 整理成**场景列表**：文件路径、使用方式（Vue template / JSX）、关键配置差异（如 `editable=true` vs `false`）

### 阶段 A3：用户确认

向用户展示分析结果，让用户确认：

```markdown
## 分析结果

**组件名**：EditableTags
**路径**：`src/components/EditableTags.vue`
**导出**：`import { EditableTags } from '@/components'`
**复杂度**：简单

**Props**：modelValue (string[]), allTags ({id,name}[]), editable (boolean), placeholder (string), showEditIcon (boolean)
**Events**：update:modelValue, confirm

**使用场景**（3 处）：
1. FlowList.columns.tsx — JSX，可编辑，无编辑图标
2. CustomFunction.columns.tsx — JSX，只读模式
3. BaseInfoCard.vue — Vue template，v-model + @confirm

**建议 Skill 名**：`use-editable-tags`
**建议触发关键词**：标签编辑、flow_tag、labels、标签列、行内标签编辑

请确认以上内容，或补充描述和触发关键词。
```

用户确认后再进入生成阶段。

### 阶段 A4：生成文件

在 `.cursor/skills/use-{name}/` 下创建文件。

#### SKILL.md 模板（复杂组件）

```markdown
---
name: use-{component-kebab-name}
description: bk-base-v4 用 {组件名}（{功能简述}）；触发：{触发关键词列表}。
---

# {组件名}

{概述段落：组件用途、适用/不适用场景}

## 真实参考

| 场景 | 路径（相对 `packages/bk-base-v4/src/`） |
|------|------|
| {场景1} | {文件路径1} |
| {场景2} | {文件路径2} |

## 入口检查清单

1. {检查项1}
2. {检查项2}
...

## Step 1：导入与基础用法

```ts
import { {组件名} } from '@/components';
```

## Step 2：Props 详解

| Prop | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| {prop1} | {type} | {default} | {说明} |

## Step 3：Events / Slots / Expose

...

## Step N：内部依赖

说明组件内部依赖了哪些子组件及其作用。

## 禁止

- {反模式1}
- {反模式2}

## 进阶（见 examples.md）

- {示例列表}
```

#### SKILL.md 模板（简单组件）

```markdown
---
name: use-{component-kebab-name}
description: bk-base-v4 用 {组件名}（{功能简述}）；触发：{触发关键词}。
---

# {组件名}

1. `import { {组件名} } from '@/components'`
2. `{prop1}: {类型}` — {说明}；默认 `{默认值}`。
3. `{prop2}?: {类型}` — {说明}；默认 `{默认值}`。
...

参考：`{组件源码路径}`、`{使用场景文件路径列表}`
```

#### examples.md 模板

```markdown
# use-{component-name} 示例

路径相对 `packages/bk-base-v4/src/`。

## 1. 标准用法

{完整 Vue template + script + style 代码块}

真实落地：{文件路径列表}

## 2. {场景名1}

{代码片段}

## 3. {场景名2}

{代码片段}

## N. 禁止：反例对照

{反例代码 + 说明}
```

生成后提示用户审查，根据反馈调整。

---

## 工作流 B：更新已有 Skill

### 阶段 B1：检测 Skill 是否存在

1. 根据组件名推导 Skill 目录名：`use-{component-kebab-name}`
2. 检查 `.cursor/skills/use-{name}/SKILL.md` 是否存在

### 阶段 B2：对比差异

1. 读取已有 `SKILL.md` 和 `examples.md`（如果存在）
2. 读取组件当前源码
3. 对比差异点：
   - **Props**：新增、删除、类型变更、默认值变更
   - **Events**：新增、删除、参数变更
   - **Slots/Expose**：新增、删除、变更
   - **内部依赖**：新增或移除的子组件

### 阶段 B3：搜索使用场景变化

1. 重新搜索所有 import 该组件的位置
2. 与 Skill 文档中记录的参考文件对比
3. 标注：新增使用位置、已移除的使用位置

### 阶段 B4：报告差异清单

向用户展示结构化的差异报告：

```markdown
## 差异报告：use-editable-tags

### 新增 Props
| Prop | 类型 | 默认值 |
|------|------|--------|
| maxTags | number | 10 |

### 删除 Props
| Prop |
|------|
| showEditIcon |

### 修改 Props
| Prop | 旧类型 | 新类型 |
|------|--------|--------|
| allTags | {id,name}[] | string[] |

### 新增 Events
| Event | 参数 |
|-------|------|
| @exceed | (count: number) |

### 新增使用场景
| 文件 | 方式 |
|------|------|
| NewPage.vue | v-model + allTags |

### 移除使用场景
| 文件 |
|------|
| OldPage.vue（组件已删除） |

请确认是否按以上变更更新 Skill 文档。
```

### 阶段 B5：增量更新

用户确认后，执行增量更新：

1. **增量原则**：
   - 替换与源码 API 直接相关的部分（Props 表、Events 表、导入语句）
   - 保留用户手写的叙事性内容（注意事项、禁止项、场景描述）
   - 新增的 API 追加到对应章节
   - 删除的 API 从文档中移除

2. **更新 SKILL.md**：
   - 更新 frontmatter 的 description（如触发关键词有变化）
   - 更新真实参考表（增减使用场景文件）
   - 更新 Props/Events 等 API 说明
   - 如新增场景，更新参考文件路径

3. **更新 examples.md**（如果存在）：
   - 新增使用场景的代码示例
   - 移除已不存在的场景示例
   - 更新标准用法中过时的 API 调用

4. 更新完成后提示用户审查。

---

## 禁止

- 跳过用户确认阶段直接生成/更新文件
- 更新时全量重写覆盖用户手写内容
- 简单组件强行套用复杂模板（或反之）
- 忘记在 SKILL.md 末尾添加参考文件路径
- 生成的 Skill 目录名不与 frontmatter name 一致
- 对不在 `@/components` 下的模块内部组件也生成独立 Skill（应归入对应模块的文档）

## 参考

- 复杂 Skill 模板：`.cursor/skills/use-bkbase-list-table/SKILL.md` + `examples.md`
- 简单 Skill 模板：`.cursor/skills/use-editable-tags/SKILL.md`、`.cursor/skills/use-custom-radio-group/SKILL.md`
- 治理规范：`.cursor/rules/cursor-governance.mdc`
