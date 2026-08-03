---
name: learn-lesson
description: 功能开发完成后调用。分析本次开发经验，判断应记录为轻量经验（写入 lessons-learned.md）还是值得沉淀为 Skill / Rule / Subagent，并引导开发者通过对应 command 创建。
---

# 开发经验总结与分级归档

## 触发时机

开发者完成功能开发、修复复杂 bug、或经历了有价值的调试过程后调用。

---

## 执行流程

### Step 1：收集上下文

**1.1 获取变更范围**

```bash
git log --oneline -15
git diff --stat
git diff --cached --stat
```

根据输出识别：涉及哪些模块、哪些组件、什么类型的工作（新模块/组件开发/API 对接/bug 修复等）。

**1.2 与开发者交互**

> 本次开发中：
> 1. 有没有遇到**意料之外**的问题？
> 2. 有没有**反复修改**了多次才确定的代码？
> 3. 有没有发现**值得复用**的模式或技巧？
> 4. 其他想记录的？
>
> 没有特别想说的也可以，我会根据代码变更自行分析。

---

### Step 2：读取现有知识体系

**2.1 经验库**（learn-lesson Skill 本身需要全量读取来做去重和升级检查）：

读取 `src/dataweb/web/packages/bk-base-v4/docs/ai-rules/lessons-learned.md`
读取 `src/dataweb/web/packages/bk-base-v4/docs/ai-rules/lessons-index.md`

> 注意：日常开发时经验库通过 fast subagent 语义搜索读取（见 `frontend-web.mdc`），
> 但本 Skill 是经验库的维护者，需要全量读取以完成去重、合并和升级检查。
> `lessons-index.md` 是日常开发的轻量入口，写入或合并经验后必须同步更新。

**2.2 已有 Skills / Rules**：

遍历 `.cursor/skills/` 和 `.cursor/rules/` 目录，了解已有哪些 Skill 和 Rule，避免建议创建已存在的。

---

### Step 3：分析提炼

基于变更内容和开发者反馈，提炼值得记录的经验。

**提炼标准（必须全部满足）**：

1. **项目特有**：是 bk-base-v4 / bk-weweb 微前端体系特有的，不是通用编程常识
2. **可操作**：能明确指导"遇到 X 场景时，应该/不应该做 Y"
3. **非冗余**：不是已有 Rule / Skill / lessons-learned.md 中已涵盖的
4. **可复现**：后续开发中可能再次遇到

如果本次没有符合标准的经验，如实告知，不强凑。

---

### Step 4：分级判断（核心）

对每条提炼出的经验，判断它属于哪个级别。

#### 判断决策树

```
这条经验涉及的场景，未来其他开发者会反复遇到吗？
│
├─ 很少，比较特定 → Level 1: 写入 lessons-learned.md
│
└─ 会反复遇到 →
    │
    这条经验能否形成一套可复用的操作流程？
    │  （比如：使用某个复杂组件、对接某类 API、配置某个工具）
    │
    ├─ 是，且流程有多个步骤，涉及特定的配置/写法/模板 →
    │  │
    │  这个流程是否复杂到需要 agent 自主编排多步操作？
    │  │
    │  ├─ 是（如：脚手架生成、多文件联动修改、自动化测试编排）→ Level 4: 建议 /create-subagent
    │  └─ 否，指导 AI 按步骤做就行 → Level 3: 建议 /create-skill
    │
    └─ 不是流程，而是一条应始终遵守的约束/规范 →
       │
       Level 2: 建议 /create-rule
```

#### 各级别判断标准详细说明

**Level 1 — 写入 lessons-learned.md**

适用于：
- 一次性的踩坑记录（某个 API 返回格式与文档不一致）
- 小范围的注意事项（某个 CSS 在微前端下的特殊表现）
- 尚未积累到可总结为模式的零散经验

**Level 2 — 建议 `/create-rule`**

适用于：
- 发现了一条应该**始终遵守**的编码约束（如"在 bk-weweb 下禁止使用 window.location.href，必须用 router"）
- 某个写法在本项目中**总是**比另一个好（如"表单校验统一使用 X 方式"）
- 已有多条 lessons-learned 指向同一个根因，适合提炼为规则

判断信号：你在分析中发现自己想说"以后所有 XXX 都应该 YYY"

**Level 3 — 建议 `/create-skill`**

适用于：
- 使用了某个**复杂组件**（表格、表单、图表等），涉及大量配置项、插槽、事件处理，且形成了一套经过验证的用法
- 对接了某类**复杂 API 模式**（分页/筛选/批量操作等），形成了可复用的实现模板
- 某个**开发场景**需要特定的多步操作流程（如"创建带权限控制的页面"）

判断信号：你在分析中发现"如果下次再做类似的事，最好按照这个步骤来"，且步骤不少于 3 步

**Level 4 — 建议 `/create-subagent`**

适用于：
- 需要 agent 自主完成**跨多个文件的联动修改**（如"根据 API 文档自动生成 Service + 类型 + Store"）
- 需要**编排多个工具调用**（如"分析组件使用情况 → 检查性能 → 生成优化方案"）
- 任务本身就是一个**端到端的自动化流程**

判断信号：你发现这个任务即使写成 Skill，AI 执行时仍然需要大量的自主判断和多轮操作

---

### Step 5：执行归档

#### Level 1：写入 lessons-learned.md

在 `lessons-learned.md` 对应分类章节末尾追加，格式：

```markdown
#### 简短标题（不超过 20 字） `#tag1` `#tag2` `#tag3`

- **日期**: YYYY-MM-DD
- **模块**: 涉及的模块名
- **场景**: 什么情况下会遇到（1 句话）
- **经验**: 该怎么做或不该怎么做（1-3 句话）
- **示例**（可选，仅文字难以表达时添加）:

  ```typescript
  // ❌ 错误做法
  ...

  // ✅ 正确做法
  ...
  ```

- **关联**（可选）: 关联条目标题 / 关联文件路径
```

**Tags 规范**：

- 放在标题末尾，用反引号包裹，以 `#` 开头
- 用于 fast subagent 语义搜索时提高命中率
- 选取 3-5 个最能描述此条目的关键词
- 常见 tags 参考：`#routing` `#bk-weweb` `#micro-frontend` `#i18n` `#navigation` `#pinia` `#table` `#form` `#component` `#api` `#auth` `#performance` `#css` `#webpack` `#type`

**去重策略**：

| 情况 | 处理 |
|------|------|
| 完全重复 | 跳过，告知已存在 |
| 部分重叠 | 合并到已有条目，追加 "(YYYY-MM-DD 补充)" |
| 同主题不同角度 | 新增条目，互相添加"关联"引用 |

写入后更新文件顶部的"最后更新"日期和"条目总数"。

同时更新 `lessons-index.md`：

- 顶部"最后更新"与"条目总数"必须与 `lessons-learned.md` 保持一致
- 在对应分类章节追加或更新一条轻量索引
- 索引条目只包含：标题 + tags、模块、1 句话场景/经验摘要
- 不复制完整示例；需要细节时由日常检索流程再读取 `lessons-learned.md` 原文
- 如果合并到已有经验，必须同步修改索引中的摘要或 tags

#### Level 2-4：引导使用对应 command

不要自行创建 Skill / Rule / Subagent，而是向开发者输出完整的创建建议。

建议中**必须包含规范约束**，因为 `/create-rule` 等 command 不会自动了解项目规范（治理规范定义在 `.cursor/rules/cursor-governance.mdc`）。

**Level 2（Rule）输出格式**：

```
💡 建议创建 Rule

主题：{主题名}
原因：{为什么需要}

创建规范：
- 文件名：`web-v4-{topic}.mdc`（bk-base-v4 相关）或 `frontend-{topic}.mdc`（前端通用）
- globs：`src/dataweb/web/packages/bk-base-v4/**`（或其他合适的路径）
- alwaysApply：false
- 内容不超过 120 行，详细内容写在 `docs/ai-rules/{topic}.md` 并引用

核心内容：
- {约束/规范 1}
- {约束/规范 2}
- {约束/规范 3}

👉 请使用 /create-rule 来创建
```

**Level 3（Skill）输出格式**：

```
💡 建议创建 Skill

主题：{主题名}
原因：{为什么需要}

创建规范：
- 目录名：`.cursor/skills/{action}-{object}/SKILL.md`，kebab-case
- 必须包含分步执行流程（Step 1, 2, 3...）
- 引用真实参考文件（项目中实际代码路径）

核心步骤：
1. {步骤 1}
2. {步骤 2}
3. {步骤 3}

关键代码模板/配置：
{如有可复用的代码片段，在此列出}

👉 请使用 /create-skill 来创建
```

**Level 4（Subagent）输出格式**：

```
💡 建议创建 Subagent

主题：{主题名}
原因：{为什么需要 agent 自主编排}

创建规范：
- 目录名：`.cursor/skills/{action}-{object}/`，kebab-case

编排流程：
1. {agent 自主完成的步骤 1}
2. {agent 自主完成的步骤 2}
3. {agent 自主完成的步骤 3}

输入/输出定义：
- 输入：{agent 需要什么信息}
- 输出：{agent 产出什么结果}

👉 请使用 /create-subagent 来创建
```

如果同时有 Level 1 和更高级别的经验，先完成 Level 1 的写入，再输出更高级别的建议。

---

### Step 6：升级检查

写入 Level 1 条目后，扫描 `lessons-learned.md`，检查是否有积累到应该升级的主题：

**升级触发条件（满足任一）**：
- 同一主题下有 **3 条及以上**密切相关的经验
- 某条经验涉及**多个模块**都可能遇到
- 某条经验涉及**安全性或数据丢失风险**

触发后，按 Level 2-4 的标准判断应升级为什么，输出建议。
如果开发者同意升级且完成了创建，回到 `lessons-learned.md` 在原条目上标注：`→ 已升级，见 .cursor/skills/{name}` 或 `→ 已升级，见 .cursor/rules/{name}.mdc`。

---

### Step 7：容量管理

写入后检查条目总数：
- **≤ 50 条**：无需处理
- **> 50 条**：提醒清理——移除已标注"已升级"的条目，将超过 6 个月未被引用的条目归档到 `docs/ai-rules/lessons-learned-archive.md`

---

## 输出汇报

```
📝 经验总结完成

变更范围：{模块/文件概述}

Level 1（写入经验库）：{N} 条
  - {标题 1}（分类：{分类}）
  - {标题 2}（分类：{分类}）

Level 2-4（建议创建）：{N} 条
  {逐条输出建议，格式见 Step 5}

跳过（已存在）：{N} 条

经验库状态：共 {总数} 条
{如有升级检查结果，在此列出}
```
