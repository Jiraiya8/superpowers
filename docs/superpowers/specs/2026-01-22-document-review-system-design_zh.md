# 文档审查系统设计

## 概述

向 superpowers 工作流添加两个新的审查阶段：

1. **规范文档审查** - 头脑风暴后，writing-plans 前
2. **计划文档审查** - writing-plans 后，实现前

两者都遵循实现审查使用的迭代循环模式。

## 规范文档审查者

**目的：** 验证规范完整、一致且准备好进行实现计划。

**位置：** `skills/brainstorming/spec-document-reviewer-prompt.md`

**检查内容：**

| 类别 | 查找项 |
|----------|------------------|
| 完整性 | TODOs、placeholders、"TBD"、不完整部分 |
| 覆盖度 | 缺失的错误处理、边界情况、集成点 |
| 一致性 | 内部矛盾、冲突的需求 |
| 清晰度 | 模糊的需求 |
| YAGNI | 未请求的功能、过度工程 |

**输出格式：**
```
## Spec Review

**Status:** Approved | Issues Found

**Issues（if any）：**
- [Section X]: [issue] - [why it matters]

**Recommendations（advisory）：**
- [suggestions that don't block approval]
```

**审查循环：** Issues found -> brainstorming agent fixes -> re-review -> repeat until approved。

**分发机制：** 使用 Task 工具，`subagent_type: general-purpose`。审查者 prompt 模板提供完整 prompt。brainstorming 技能的 controller 分发审查者。

## 计划文档审查者

**目的：** 验证计划完整、匹配规范、有正确的任务分解。

**位置：** `skills/writing-plans/plan-document-reviewer-prompt.md`

**检查内容：**

| 类别 | 查找项 |
|----------|------------------|
| 完整性 | TODOs、placeholders、不完整任务 |
| 规范对齐 | 计划覆盖规范需求，无范围蔓延 |
| 任务分解 | 任务原子、边界清晰 |
| 任务语法 | 任务和步骤使用 checkbox 语法 |
| Chunk 大小 | 每个 chunk 低于 1000 行 |

**Chunk 定义：** Chunk 是计划文档中任务的逻辑分组，由 `## Chunk N: <name>` 标题分隔。writing-plans 技能基于逻辑阶段创建这些边界（例如 "Foundation"、"Core Features"、"Integration"）。每个 chunk 应足够自包含以独立审查。

**规范对齐验证：** 审查者接收两者：
1. 计划文档（或当前 chunk）
2. 规范文档路径供参考

审查者阅读两者并比较需求覆盖。

**输出格式：** 与规范审查者相同，但范围限定到当前 chunk。

**审查流程（逐 chunk）：**
1. Writing-plans 创建 chunk N
2. Controller 分发 plan-document-reviewer，带 chunk N 内容和规范路径
3. 审查者阅读 chunk 和规范，返回结论
4. 如有问题：writing-plans agent 修复 chunk N，转到步骤 2
5. 如批准：继续 chunk N+1
6. 重复直到所有 chunk 批准

**分发机制：** 与规范审查者相同 — Task 工具，`subagent_type: general-purpose`。

## 更新的工作流

```
brainstorming -> spec -> SPEC REVIEW LOOP -> writing-plans -> plan -> PLAN REVIEW LOOP -> implementation
```

**规范审查循环：**
1. 规范完成
2. 分发审查者
3. 如有问题：修复 -> 转到 2
4. 如批准：继续

**计划审查循环：**
1. Chunk N 完成
2. 为 chunk N 分发审查者
3. 如有问题：修复 -> 转到 2
4. 如批准：下一个 chunk 或实现

## Markdown 任务语法

任务和步骤使用 checkbox 语法：

```markdown
- [ ] ### Task 1: Name

- [ ] **Step 1:** Description
  - File: path
  - Command: cmd
```

## 错误处理

**审查循环终止：**
- 无硬性迭代限制 — 循环继续直到审查者批准
- 如果循环超过 5 次迭代，controller 应向人类提示指导
- 人类可选择：继续迭代、批准已知问题、或中止

**分歧处理：**
- 审查者是建议性的 — 他们标记问题但不阻塞
- 如果 agent 认为审查者反馈不正确，应在修复中解释原因
- 如果同一问题 3 次迭代后仍有分歧，向人类提示

**畸形审查者输出：**
- Controller 应验证审查者输出有必需字段（Status、Issues if applicable）
- 如畸形，重新分发审查者并注明期望格式
- 2 次畸形响应后，向人类提示

## 需变更的文件

**新文件：**
- `skills/brainstorming/spec-document-reviewer-prompt.md`
- `skills/writing-plans/plan-document-reviewer-prompt.md`

**修改文件：**
- `skills/brainstorming/SKILL.md` - 规范写入后添加审查循环
- `skills/writing-plans/SKILL.md` - 添加逐 chunk 审查循环，更新任务语法示例