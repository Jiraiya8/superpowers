# 文档审查系统实现计划

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development（如有子代理）或 superpowers:executing-plans 实现此计划。

**目标：** 向 brainstorming 和 writing-plans 技能添加规范和计划文档审查循环。

**架构：** 在每个技能目录创建审查者 prompt 模板。修改技能文件在文档创建后添加审查循环。使用 Task 工具，general-purpose 子代理分发审查者。

**技术栈：** Markdown 技能文件、通过 Task 工具的子代理分发

**规范：** docs/superpowers/specs/2026-01-22-document-review-system-design.md

---

## Chunk 1: 规范文档审查者

此 chunk 向 brainstorming 技能添加规范文档审查者。

### Task 1: 创建规范文档审查者 Prompt 模板

**文件：**
- 创建：`skills/brainstorming/spec-document-reviewer-prompt.md`

- [ ] **Step 1:** 创建审查者 prompt 模板文件

```markdown
# 规格文档审查者提示模板

在派发规格文档审查者子代理时使用此模板。

**目的：** 验证规格完整、一致且准备好进行实现规划。

**派发时机：** 规格文档已写入 docs/superpowers/specs/

```
Task tool（general-purpose）:
  description: "审查规格文档"
  prompt: |
    你是一个规格文档审查者。验证此规格完整且准备好进行规划。

    **待审查规格：** [SPEC_FILE_PATH]

    ## 检查内容

    | 类别 | 查找内容 |
    |----------|------------------|
    | 完整性 | TODO、占位符、"TBD"、不完整章节 |
    | 覆盖率 | 缺失的错误处理、边界情况、集成点 |
    | 一致性 | 内部矛盾、冲突需求 |
    | 清晰度 | 模糊的需求 |
    | YAGNI | 未请求的功能、过度工程 |

    ## 关键

    特别仔细检查：
    - 任何 TODO 标记或占位符文本
    - 说"稍后定义"或"等 X 完成后再规格化"的章节
    - 明显比其他章节不够详细的章节

    ## 输出格式

    ## 规格审查

    **状态：** ✅ 已批准 | ❌ 发现问题

    **问题（如有）：**
    - [章节 X]: [具体问题] - [为什么重要]

    **建议（仅供参考）：**
    - [不阻止批准的建议]
```

**审查者返回：** 状态、问题（如有）、建议
```

- [ ] **Step 2:** 验证文件创建正确

运行：`cat skills/brainstorming/spec-document-reviewer-prompt.md | head -20`
预期：显示 header 和 purpose 部分

- [ ] **Step 3:** 提交

```bash
git add skills/brainstorming/spec-document-reviewer-prompt.md
git commit -m "feat: add spec document reviewer prompt template"
```

---

### Task 2: 向 Brainstorming 技能添加审查循环

**文件：**
- 修改：`skills/brainstorming/SKILL.md`

- [ ] **Step 1:** 读取当前 brainstorming 技能

运行：`cat skills/brainstorming/SKILL.md`

- [ ] **Step 2:** 在 "After the Design" 后添加审查循环部分

找到 "After the Design" 部分，在 documentation 后、implementation 前添加新的 "Spec Review Loop" 部分：

```markdown
**规格审查循环：**
编写规格文档后：
1. 派发 spec-document-reviewer 子代理（见 spec-document-reviewer-prompt.md）
2. 如果 ❌ 发现问题：
   - 在规格文档中修复问题
   - 重新派发审查者
   - 重复直到 ✅ 已批准
3. 如果 ✅ 已批准：继续到实现设置

**审查循环指导：**
- 编写规格的同一代理修复它（保留上下文）
- 如果循环超过 5 次，向用户请求指导
- 审查者仅供参考 — 如果你认为反馈错误，解释你的不同意见
```

- [ ] **Step 3:** 验证变更

运行：`grep -A 15 "Spec Review Loop" skills/brainstorming/SKILL.md`
预期：显示新审查循环部分

- [ ] **Step 4:** 提交

```bash
git add skills/brainstorming/SKILL.md
git commit -m "feat: add spec review loop to brainstorming skill"
```

---

## Chunk 2: 计划文档审查者

此 chunk 向 writing-plans 技能添加计划文档审查者。

### Task 3: 创建计划文档审查者 Prompt 模板

**文件：**
- 创建：`skills/writing-plans/plan-document-reviewer-prompt.md`

- [ ] **Step 1:** 创建审查者 prompt 模板文件

```markdown
# 计划文档审查者提示模板

在派发计划文档审查者子代理时使用此模板。

**目的：** 验证计划块完整、匹配规格、并有合理的任务分解。

**派发时机：** 每个计划块编写完成后

```
Task tool（general-purpose）:
  description: "审查计划块 N"
  prompt: |
    你是一个计划文档审查者。验证此计划块完整且准备好进行实现。

    **待审查计划块：** [PLAN_FILE_PATH] - 仅块 N
    **参考规格：** [SPEC_FILE_PATH]

    ## 检查内容

    | 类别 | 查找内容 |
    |----------|------------------|
    | 完整性 | TODO、占位符、不完整任务、缺失步骤 |
    | 规格一致性 | 块覆盖相关规格需求，无范围蔓延 |
    | 任务分解 | 任务原子化、边界清晰、步骤可操作 |
    | 任务语法 | 任务和步骤使用复选框语法（`- [ ]`） |
    | 块大小 | 每块不超过 1000 行 |

    ## 关键

    特别仔细检查：
    - 任何 TODO 标记或占位符文本
    - 说"类似于 X"但没有实际内容的步骤
    - 不完整的任务定义
    - 缺失的验证步骤或预期输出

    ## 输出格式

    ## 计划审查 - 块 N

    **状态：** ✅ 已批准 | ❌ 发现问题

    **问题（如有）：**
    - [Task X，Step Y]: [具体问题] - [为什么重要]

    **建议（仅供参考）：**
    - [不阻止批准的建议]
```

**审查者返回：** 状态、问题（如有）、建议
```

- [ ] **Step 2:** 验证文件创建

运行：`cat skills/writing-plans/plan-document-reviewer-prompt.md | head -20`
预期：显示 header 和 purpose 部分

- [ ] **Step 3:** 提交

```bash
git add skills/writing-plans/plan-document-reviewer-prompt.md
git commit -m "feat: add plan document reviewer prompt template"
```

---

### Task 4: 向 Writing-Plans 技能添加审查循环

**文件：**
- 修改：`skills/writing-plans/SKILL.md`

- [ ] **Step 1:** 读取当前技能文件

运行：`cat skills/writing-plans/SKILL.md`

- [ ] **Step 2:** 添加逐 chunk 审查部分

在 "Execution Handoff" 部分前添加：

```markdown
## 计划审查循环

完成计划的每个块后：

1. 为当前块派发 plan-document-reviewer 子代理
   - 提供：块内容、规格文档路径
2. 如果 ❌ 发现问题：
   - 在块中修复问题
   - 为该块重新派发审查者
   - 重复直到 ✅ 已批准
3. 如果 ✅ 已批准：继续到下一块（或如果是最后一块，继续到执行移交）

**块边界：** 使用 `## Chunk N: <名称>` 标题分隔块。每块应 ≤1000 行且逻辑上自包含。
```

- [ ] **Step 3:** 更新任务语法示例使用 checkbox

更改 Task Structure 部分显示 checkbox 语法：

```markdown
### Task N: [Component Name]

- [ ] **Step 1:** Write the failing test
  - File: `tests/path/test.py`
  ...
```

- [ ] **Step 4:** 验证审查循环部分已添加

运行：`grep -A 15 "Plan Review Loop" skills/writing-plans/SKILL.md`
预期：显示新审查循环部分

- [ ] **Step 5:** 验证任务语法示例已更新

运行：`grep -A 5 "Task N:" skills/writing-plans/SKILL.md`
预期：显示 checkbox 语法 `### Task N:`

- [ ] **Step 6:** 提交

```bash
git add skills/writing-plans/SKILL.md
git commit -m "feat: add plan review loop and checkbox syntax to writing-plans skill"
```

---

## Chunk 3: 更新计划文档 Header

此 chunk 更新计划文档 header 模板以引用新的 checkbox 语法要求。

### Task 5: 更新 Writing-Plans 技能中的计划 Header 模板

**文件：**
- 修改：`skills/writing-plans/SKILL.md`

- [ ] **Step 1:** 读取当前计划 header 模板

运行：`grep -A 20 "Plan Document Header" skills/writing-plans/SKILL.md`

- [ ] **Step 2:** 更新 header 模板引用 checkbox 语法

计划 header 应注明任务和步骤使用 checkbox 语法。更新 header comment：

```markdown
> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development（if subagents available）or superpowers:executing-plans to implement this plan。Tasks and steps use checkbox（`- [ ]`）syntax for tracking。
```

- [ ] **Step 3:** 验证变更

运行：`grep -A 5 "For agentic workers:" skills/writing-plans/SKILL.md`
预期：显示更新的 header，包含 checkbox 语法提及

- [ ] **Step 4:** 提交

```bash
git add skills/writing-plans/SKILL.md
git commit -m "docs: update plan header to reference checkbox syntax"
```